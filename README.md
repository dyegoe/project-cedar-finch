# Project Cedar Finch

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
