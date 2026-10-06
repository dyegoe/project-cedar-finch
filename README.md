# Project Cedar Finch

Project Cedar Finch demonstrates modernizing a legacy PHP web service for cloud-native deployment: it packages the stateless app in a minimal, non-root FrankenPHP container with health endpoints and standard-stream logging, and deploys it through a Helm chart with Kubernetes configuration, secrets, probes, and CPU-based autoscaling. The chart targets the Gateway API and can be run locally on kind with Podman and Envoy Gateway, so the service can be built and verified without an AWS account.

## Decisions

- **Containerization:** The legacy PHP service is packaged in a single, minimal FrankenPHP container to simplify deployment and operations.
- **Non-root execution:** The container runs as a non-root user (`www-data`, uid 82) to enhance security.
- **Port selection:** The service listens on port 8080, an unprivileged port suitable for non-root users.
- **No TLS in the container:** TLS termination is expected to be handled by an ingress or load balancer, keeping the container simple.
- **Standard-stream logging:** Access and error logs are written to stdout and stderr as JSON, avoiding log file management.
- **Health endpoints:** Liveness and readiness probes are exposed at `/healthz` and `/readyz` for Kubernetes to monitor the service.
- **Helm deployment:** The service is deployed using a Helm chart with configurable values, secrets, and CPU-based autoscaling.
- **Gateway API:** The chart targets the Gateway API, allowing integration with modern ingress solutions like Envoy Gateway instead of the older Ingress API.

## Containerized PHP service

The PHP service in [`app/index.php`](app/index.php) is packaged by [`app/Dockerfile`](app/Dockerfile) and configured by [`app/Caddyfile`](app/Caddyfile).

### Build and run

```sh
docker build -t legacy-web-service:local app
docker run --rm -p 8080:8080 legacy-web-service:local
```

```sh
curl -i localhost:8080/          # service info
curl -i localhost:8080/healthz   # liveness, HTTP 200
curl -i localhost:8080/readyz    # readiness, HTTP 200
```

The service reads `APP_NAME`, `APP_ENV`, `DB_HOST`, `DB_PORT` and `CACHE_HOST` from the environment. For example: `docker run -e APP_ENV=staging ...`.

### Why FrankenPHP

The service is a small, stateless HTTP endpoint, so I used a single container with [FrankenPHP](https://frankenphp.dev) (Caddy with embedded PHP), based on its Alpine image.

- **One process, one image.** There is no supervisor and no second container. This keeps the operational model simple: one foreground process that Kubernetes can start, probe and stop.
- **Simple logging.** Caddy and PHP write directly to the container's standard streams.
- **Why not PHP-FPM plus a web server.** That split pays off when PHP and the web layer need separate lifecycles, tuning, scaling or ownership. None of those apply here, and it would add a second container or a supervisor for no benefit.

### Runtime decisions

- **Non-root.** The image runs as `www-data` (uid 82). Caddy's state directories (`/data/caddy`, `/config/caddy`) are made writable for that user.
- **Port 8080.** An unprivileged port, because non-root users can't bind to ports below 1024.
- **No TLS in the container.** `auto_https` is off and the admin API is disabled. TLS termination is expected to happen at an ingress or load balancer. As a result only HTTP/1.1 is served.
- **Logs on standard streams only.** Access logs go to stdout and runtime/error logs go to stderr, both as JSON. No log files are written. Read them with `docker logs`. To check the split, discard one stream at a time:

  ```sh
  docker run --rm -p 8080:8080 legacy-web-service:local 2>/dev/null   # stdout only: access logs
  docker run --rm -p 8080:8080 legacy-web-service:local >/dev/null    # stderr only: runtime logs
  ```

- **Verify the user.** `docker exec <container> id` should show `uid=82(www-data)`.

### Notes

- `/readyz` returns 200 without checking the database or cache. The supplied service only reads their host settings, so no dependencies were added.

## Run the Helm chart on kind with Podman

The chart is cluster-agnostic: it creates a Gateway API `HTTPRoute` that attaches to an existing `Gateway` (`httpRoute.parentRefs`, default `eg` in `default`). The kind-specific setup lives in [kind/](kind/) and uses [Envoy Gateway](https://gateway.envoyproxy.io/).

Install [kind](https://kind.sigs.k8s.io/), Helm and kubectl, then:

```sh
export KIND_EXPERIMENTAL_PROVIDER=podman

# 1. Cluster: maps host port 8080 to NodePort 30080
kind create cluster --name cedar-finch --config kind/kind-config.yaml

# 2. Envoy Gateway (also installs the Gateway API CRDs)
helm install eg oci://docker.io/envoyproxy/gateway-helm --version v1.5.0 \
  -n envoy-gateway-system --create-namespace
kubectl wait --timeout=3m -n envoy-gateway-system deployment/envoy-gateway --for=condition=Available

# 3. GatewayClass + Gateway exposed as NodePort 30080
kubectl apply -f kind/gateway.yaml

# 4. Build, load and install the service
podman build -t localhost/legacy-web-service:local app
kind load docker-image localhost/legacy-web-service:local --name cedar-finch
helm install legacy-web-service helm \
  --set image.repository=localhost/legacy-web-service \
  --set image.tag=local \
  --set image.pullPolicy=IfNotPresent --wait
```

Verify (no port-forward needed):

```sh
curl -i localhost:8080/healthz
curl -i localhost:8080/readyz
curl -i localhost:8080/
```

The Gateway listener and the `HTTPRoute` have no hostname, so they match every host. This was a deliberate choice to avoid needing an `/etc/hosts` entry locally; set `httpRoute.hostnames` (and a listener hostname) for real environments. On another cluster, point `httpRoute.parentRefs` at your own Gateway, or set `httpRoute.enabled=false`.

The chart creates a CPU-based HPA; live scaling metrics require metrics-server in the cluster. Clean up with `kind delete cluster --name cedar-finch`.
