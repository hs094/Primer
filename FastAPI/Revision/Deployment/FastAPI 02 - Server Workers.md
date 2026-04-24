# Dep 02 — Run a Server & Workers

## ASGI servers

- **Uvicorn** — most common; the FastAPI CLI runs this.
- **Hypercorn** — HTTP/2 + HTTP/3 support.
- **Daphne** — ties in Django Channels ecosystem.

## Launch (dev)

```bash
fastapi dev main.py
uvicorn main:app --reload
```

## Launch (prod)

```bash
fastapi run main.py                   # host 0.0.0.0, no reload, 1 worker
uvicorn main:app --host 0.0.0.0 --port 8000 --workers 4
```

Flags worth knowing:

| Flag | Purpose |
|---|---|
| `--workers N` | spawn N worker processes |
| `--proxy-headers` | trust `X-Forwarded-*` |
| `--forwarded-allow-ips="*"` | who's allowed to send them |
| `--root-path /api` | behind a prefix proxy |
| `--log-level info` | `debug` / `info` / `warning` / `error` |
| `--access-log` / `--no-access-log` | per-request logs |
| `--timeout-keep-alive 5` | keepalive seconds |

## Gunicorn + Uvicorn workers (classic setup)

```bash
gunicorn main:app \
  -k uvicorn.workers.UvicornWorker \
  --workers 4 \
  --bind 0.0.0.0:8000 \
  --timeout 60 \
  --graceful-timeout 30 \
  --keep-alive 5
```

Why: Gunicorn is a battle-tested process manager (preload, graceful restart, signal handling). In containerized setups with an external supervisor (Kubernetes), plain Uvicorn `--workers` is usually enough.

## Picking the worker count

- CPU-bound heavy: workers ≈ cores.
- Mostly async I/O: fewer workers suffice (each can serve many concurrent requests).
- One-process-per-container is also fine; scale via replicas at the orchestrator level.

## Shared state warning

Each worker is its own Python interpreter.
- In-memory caches, rate limiters, WebSocket registries don't share across workers.
- For cross-worker coordination: Redis, database, or a message bus.

## Gotchas

- ⚠️ `uvicorn --workers N` with `--reload` is not supported.
- ⚠️ Don't start `--workers` inside Kubernetes pods if you're also scaling replicas — prefer 1 worker per pod and let the orchestrator do the replication.
- ⚠️ `--timeout-keep-alive` lower than your LB's idle timeout → connection resets.
