# Dep 01 — Deployment Concepts

🔑 What "deploying" actually requires: security (TLS), process supervision, replicas, memory, previous dependencies.

## The pillars

| Concern | What it means in practice |
|---|---|
| HTTPS | Terminate TLS (proxy or ASGI server with certs) — browsers and APIs need it. |
| Running on startup | The app must come back up after crashes and reboots. |
| Restarts | A supervisor restarts the process if it dies. |
| Replication (workers) | Multiple processes → parallelism across CPU cores. |
| Memory | Each worker process has its own memory. Plan for peak × workers. |
| Previous steps | Migrations, static files, cache warmup before handling traffic. |

## HTTPS

Handled outside FastAPI 99% of the time:

- **Behind a TLS-terminating proxy** (nginx, Traefik, Cloudflare, ELB) → app listens HTTP on a private port.
- **Direct** → Uvicorn with `--ssl-keyfile` / `--ssl-certfile` (less common).

Use Let's Encrypt for free certs; automate renewal (Traefik handles this out of the box).

## Process supervision

Options:

- `systemd` unit on a VM.
- Container orchestrator (Kubernetes, ECS, Nomad, Fly.io, Railway).
- `docker compose` with `restart: unless-stopped` (simple deployments).

Never rely on `uvicorn --reload` for prod.

## Replication

```
Public → TLS proxy → N app processes (workers) → DB/cache
```

- Workers = `NUM_CORES` for CPU-bound work, `2×NUM_CORES+1` rule of thumb for mixed.
- Each worker is a full Python process (separate memory, separate DB pool).

## Memory sizing

- Measure RSS per worker under load.
- DB pool size × workers must be ≤ DB `max_connections` with slack.
- Add GC, buffers, caches to your per-worker estimate.

## Previous steps

Before serving traffic:
- Run DB migrations (Alembic).
- Collect static assets if applicable.
- Warm read-only caches only when that matters.

Put migrations in a one-shot job (not every worker's `lifespan` — race conditions).

## Health checks

- `/livez` — process alive.
- `/readyz` — can serve traffic (deps ready, migrations done).
- Keep them cheap (no DB round-trip in liveness).

## Graceful shutdown

- Handle `SIGTERM`: stop accepting new requests, let in-flight finish, close pools.
- Uvicorn does this by default; don't `kill -9` in your supervisor.
