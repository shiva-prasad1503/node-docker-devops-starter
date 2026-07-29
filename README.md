# devops-app

A Node.js/Express API, fully containerized and wired up with DevOps best practices:
Docker (multi-stage, non-root), Docker Compose (app + Postgres + Redis + Prometheus + Grafana),
automated tests, CI/CD via GitHub Actions, structured logging, and metrics/monitoring.

## Stack

- **App**: Node.js 20 + Express
- **DB**: PostgreSQL 16
- **Cache**: Redis 7
- **Logging**: pino (structured JSON logs)
- **Metrics**: prom-client (`/metrics`) → Prometheus → Grafana dashboard
- **Tests**: Jest + Supertest
- **CI/CD**: GitHub Actions (lint → test → build → push to GHCR)

## Project layout

```
src/
  index.js          # entrypoint, graceful shutdown
  app.js             # express app factory (testable)
  db.js               # Postgres pool
  cache.js            # Redis client
  logger.js           # pino logger
  metrics.js           # Prometheus metrics + middleware
  routes/health.js      # /health (liveness), /ready (readiness)
  routes/items.js       # example DB-backed CRUD
  middleware/errorHandler.js
tests/                # Jest test suites
monitoring/            # Prometheus + Grafana provisioning
.github/workflows/       # CI/CD pipeline
Dockerfile              # multi-stage build (deps → test → production)
docker-compose.yml       # full local stack
```

## Run it locally

1. Copy the env file (compose already sets these itself, this is for running outside Docker):
   ```bash
   cp .env.example .env
   ```

2. Build and start everything:
   ```bash
   docker compose up --build
   ```

   This starts:
   - `app` → http://localhost:3000
   - `postgres` → localhost:5432
   - `redis` → localhost:6379
   - `prometheus` → http://localhost:9090
   - `grafana` → http://localhost:3001 (login: `admin` / `admin`)

3. Check it's healthy:
   ```bash
   curl http://localhost:3000/health     # liveness
   curl http://localhost:3000/ready      # readiness (checks DB + Redis)
   curl http://localhost:3000/metrics    # Prometheus metrics
   ```

4. Try the example API:
   ```bash
   curl -X POST http://localhost:3000/api/items -H "Content-Type: application/json" -d '{"name":"widget"}'
   curl http://localhost:3000/api/items
   ```

5. Open Grafana at http://localhost:3001 — the "DevOps App Overview" dashboard is
   auto-provisioned with request rate, p95 latency, error rate, and memory panels.

6. Tear down:
   ```bash
   docker compose down          # stop containers
   docker compose down -v       # also wipe DB/Grafana volumes
   ```

## Run without Docker (dev loop)

```bash
npm install
npm run dev          # requires local Postgres/Redis, or point env vars at remote ones
npm test              # run test suite
npm run lint          # eslint
```

## Docker image details

- Multi-stage build: `deps` (prod deps only) → `build` (installs devDeps, runs tests as a build
  gate) → `production` (final slim image, non-root `appuser`, `dumb-init` as PID 1, built-in
  `HEALTHCHECK`).
- If tests fail, the image build fails — tests are a hard gate before you can ship the image.

Build manually:
```bash
docker build -t devops-app:local --target production .
docker run -p 3000:3000 devops-app:local
```

## CI/CD pipeline (`.github/workflows/ci-cd.yml`)

On every push/PR to `main`:

1. **lint** — ESLint
2. **test** — Jest against real Postgres + Redis service containers, uploads coverage artifact
3. **build-and-push** (main branch only) — builds the production image and pushes to
   `ghcr.io/<your-org>/<repo>/devops-app` tagged with both the git SHA and `latest`

To deploy automatically after a successful build, uncomment/fill in the `deploy` job stub at the
bottom of the workflow (SSH + `docker compose pull && up -d`, `kubectl set image`, etc. — depends
on your target infra).

## Observability

- **Logs**: structured JSON via pino, written to stdout (containers assumed to be shipped to your
  log aggregator, e.g. `docker compose logs -f app`, or forwarded via a log driver in real
  deployments).
- **Metrics**: `/metrics` exposes default Node.js process metrics plus `http_requests_total` and
  `http_request_duration_seconds` (labeled by method/route/status).
- **Dashboards**: Grafana auto-provisions the Prometheus datasource and the "DevOps App Overview"
  dashboard on startup — no manual clicking required.
- **Health checks**: `/health` (liveness — is the process alive) vs `/ready` (readiness — can it
  serve traffic, i.e. are DB/Redis reachable). Compose and the Dockerfile `HEALTHCHECK` use these.

## Notes on going to production

- Replace the hardcoded Compose credentials with Docker secrets / a secrets manager.
- Point CI's `build-and-push` at your actual registry if not using GHCR.
- Add a real `deploy` job (SSH, Kubernetes, ECS, etc.) — left as a stub since it's
  infra-specific.
- Consider adding rate limiting, request validation (e.g. zod), and a WAF/reverse proxy (nginx/
  Traefik) in front of the app for real deployments.
