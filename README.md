# Project Cedar Finch

This repository modernizes a legacy PHP service for cloud-native deployment. It includes a minimal, non-root FrankenPHP container, a Helm chart, a local kind setup, and a GitHub Actions workflow for building and publishing the image and chart. The local verification path uses no AWS account or cloud resources.

## What is implemented

- The service is packaged in [`app/Dockerfile`](app/Dockerfile) and configured by [`app/Caddyfile`](app/Caddyfile). It runs as the non-root `www-data` user on port 8080; access and runtime logs go to stdout and stderr.
- The [`helm/`](helm/) chart deploys two replicas by default, with resource requests and limits, rolling updates, liveness/readiness probes, a ConfigMap, a Secret, a ClusterIP Service, and an optional CPU-based HPA.
- The chart publishes a Gateway API `HTTPRoute` that attaches to an existing `Gateway`. This uses the newer Kubernetes Gateway API rather than the older Ingress API. It is **not** Amazon API Gateway; the local implementation uses Envoy Gateway.
- [`kind/`](kind/) contains the local cluster configuration and Envoy Gateway resources.
- [`.github/workflows/publish-container.yml`](.github/workflows/publish-container.yml) builds and publishes the container and Helm chart as described below.

### Deliberate scope decisions

- **No Terraform or AWS deployment:** I deliberately did not add Terraform or provision EKS. The assignment can be built and verified locally without AWS, and deploying EKS would add cloud cost that is unnecessary for demonstrating the implemented container and Kubernetes work. For a real EKS deployment, I would add Terraform and use EKS Pod Identity or IRSA for AWS access rather than static AWS credentials. My Terraform work is represented in separate projects: [terraform-aws-base-infra](https://github.com/dyegoe/terraform-aws-base-infra), [terraform-proxmox](https://github.com/dyegoe/terraform-proxmox), and [terraform-libvirt](https://github.com/dyegoe/terraform-libvirt). These are references to my Terraform experience, not dependencies or components of this repository.
- **Gateway API instead of Ingress:** I chose the Kubernetes Gateway API and `HTTPRoute` as the more modern, extensible routing API. The chart expects a Gateway and its controller to exist; the repository supplies Envoy Gateway only for the local kind path. No AWS-specific gateway/controller configuration is included.

## Build and verify the container locally

Prerequisites: Docker (or Podman) and `curl`.

```sh
docker build -t legacy-web-service:local app
docker run --rm -d --name legacy-web-service -p 8080:8080 legacy-web-service:local

curl -i http://localhost:8080/
curl -i http://localhost:8080/healthz
curl -i http://localhost:8080/readyz

docker stop legacy-web-service
```

The root endpoint returns service information; `/healthz` and `/readyz` return HTTP 200. Read the application and access logs with `docker logs <container>`. The readiness endpoint currently checks that the service is responding; it does not test database or cache connectivity.

## Deploy and verify on kind

Prerequisites: [kind](https://kind.sigs.k8s.io/), Podman, `kubectl`, Helm, and `curl`. These commands use Podman as kind's container provider and need no AWS account.

```sh
export KIND_EXPERIMENTAL_PROVIDER=podman

# Create kind and map localhost:8080 to the cluster's Envoy Gateway.
kind create cluster --name cedar-finch --config kind/kind-config.yaml

# Install Envoy Gateway (including Gateway API CRDs) and wait for its controller.
helm install eg oci://docker.io/envoyproxy/gateway-helm --version v1.9.2 \
  --namespace envoy-gateway-system --create-namespace
kubectl wait --timeout=3m -n envoy-gateway-system \
  deployment/envoy-gateway --for=condition=Available

# Create the local GatewayClass and Gateway.
kubectl apply -f kind/gateway.yaml
kubectl wait --for=condition=Programmed --timeout=3m gateway/eg

# Build and load the same image into kind, then install the chart.
podman build -t localhost/legacy-web-service:local app
kind load docker-image localhost/legacy-web-service:local --name cedar-finch
helm install legacy-web-service helm \
  --set image.repository=localhost/legacy-web-service \
  --set image.tag=local --wait
# HTTPRoute conditions are nested under status.parents, not status.conditions.
kubectl wait \
  --for='jsonpath={.status.parents[0].conditions[?(@.type=="Accepted")].status}=True' \
  --timeout=2m httproute/legacy-web-service

# Verify the deployed endpoints through the Gateway.
curl -i http://localhost:8080/
curl -i http://localhost:8080/healthz
curl -i http://localhost:8080/readyz

# Optional: inspect the deployed resources and logs.
kubectl get pods,service,httproute,hpa
kubectl logs deployment/legacy-web-service

# Remove the local cluster when finished.
kind delete cluster --name cedar-finch
```

The chart defaults to `image.pullPolicy: IfNotPresent`, so the local install explicitly sets the repository and tag to match the image loaded into kind. If rebuilding without deleting the cluster, use a new tag and update the release with that tag so the node does not reuse a cached image.

The local Gateway has no hostname restriction and accepts routes from all namespaces to keep local testing simple. Restrict both in a real environment. HPA CPU metrics require Metrics Server; kind may not have it installed, so the HPA resource can be present without live scaling data. For ingress/routing tests, a Gateway API controller is required; the local commands install Envoy Gateway. On another cluster, configure `httpRoute.parentRefs` for an existing Gateway or disable the route with `--set httpRoute.enabled=false`.

## Image and Helm chart publishing

The GitHub Actions workflow runs on pull requests to `main` and pushes to `main`:

- Pull requests build the image without publishing it, then lint and package the chart.
- Pushes to `main` publish `ghcr.io/dyegoe/project-cedar-finch` with `sha-<full commit SHA>` and `main` tags (for example `sha-d95f1ba5aaccde3141e6e74c4d891f173612b0ba`), and publish the chart to `oci://ghcr.io/dyegoe/charts/legacy-web-service`. The workflow uses the repository's `GITHUB_TOKEN`; no registry password is committed or required.
- The published chart version is the `Chart.yaml` version with `-g<short-sha>` appended, and its `appVersion` is `sha-<full commit SHA>`. This makes the chart's default image tag match the image published from that commit.

The chart values select the image: `image.repository` chooses GHCR or the local kind image, `image.tag` selects a tag, and `image.digest` can pin an immutable digest instead of a tag. For a GHCR deployment, use an immutable commit tag or digest. If the image package is private, authenticate to GHCR and configure a Kubernetes image pull Secret through `imagePullSecrets`.

Only commits pushed to `main` are published, so your local `git rev-parse HEAD` may not match any image tag (for example, with local or unmerged commits, or after a squash merge). Select the tag from the latest published `main` commit instead:

```sh
git fetch origin main
TAG="sha-$(git rev-parse origin/main)"

helm upgrade --install legacy-web-service helm \
  --set image.repository=ghcr.io/dyegoe/project-cedar-finch \
  --set-string image.tag="$TAG" --wait
```

If the workflow for that commit is still running or failed, the tag won't exist yet. Check the workflow run and the GHCR package page for available tags, or use the floating `main` tag for a quick, non-reproducible check (`--set-string image.tag=main`; `image.pullPolicy` stays `IfNotPresent`, so a node may keep a stale image).

## Observability and production follow-ups

These are recommendations, not components configured by this repository:

- **Logs and metrics:** In EKS, collect container stdout/stderr with Fluent Bit and ship logs to CloudWatch Logs or OpenSearch. Scrape application and Kubernetes metrics with Prometheus and visualize/alert on them with Grafana. Add dashboards and alerts for availability, latency, error rate, saturation, pod restarts, and HPA behavior.
- **GitOps and delivery:** Manage environment-specific deployments with Argo CD (or Flux). Extend CI/CD with chart rendering and validation, deployment tests, rollback checks, and promotion of immutable image digests. Kargo can automate promotion between environments, and Argo Rollouts can provide progressive delivery (canary or blue/green) with automated analysis. My experience with Argo CD, Kargo, and Argo Rollouts is represented in my [homelab](https://github.com/dyegoe/homelab) repository, a separate project and not a dependency of this one.
- **Supply-chain security:** Scan images and dependencies, generate an SBOM, sign images, and enforce signature/provenance policies with admission controls.
- **Secrets and cluster security:** Integrate an external secret manager (for example AWS Secrets Manager through External Secrets Operator and EKS Pod Identity/IRSA); add restrictive network policies and admission policies; avoid storing production credentials in Helm values.
- **Resilience and operations:** Define backup and restore procedures for stateful dependencies and cluster configuration, and regularly test recovery and incident response.

The included HPA is CPU-based, but operational autoscaling requires Metrics Server and working resource metrics. Production Gateway/API integration, TLS, DNS, WAF/CDN, and AWS infrastructure provisioning are also outside the implemented local example.

## AI assistance

AI assistance was used to help organize and refine this repository documentation, including the local verification steps and the explanation of implementation decisions and production follow-ups. The repository configuration and assignment requirements were used as context. **All documentation was reviewed by a human, and the documented commands were reproduced by a human** on a local kind cluster before being kept. AI-generated suggestions are not a substitute for running the commands in an environment with the required tools installed.
