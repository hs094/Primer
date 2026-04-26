# Deploy 06 — Run a Server Manually

🔑 In production, run a real ASGI server — Uvicorn (or Hypercorn / Granian). `fastapi dev` is for local development only.

## Uvicorn — single process

```bash
uvicorn main:app --host 0.0.0.0 --port 8000
```

- `main:app` — `module:variable`.
- `--host 0.0.0.0` — bind all interfaces.
- `--port 8000` — choose a port; usually behind a reverse proxy on 80/443.

## Useful flags

```bash
uvicorn main:app \
  --host 0.0.0.0 --port 8000 \
  --workers 4 \
  --log-level info \
  --proxy-headers \
  --forwarded-allow-ips="*" \
  --timeout-graceful-shutdown 30
```

- `--workers N` — multiple processes (no shared memory). Tune to CPU count.
- `--proxy-headers` + `--forwarded-allow-ips` — trust `X-Forwarded-*` from your load balancer ([[FastAPI 08 - Sub Apps and Proxy]]).
- `--timeout-graceful-shutdown` — let in-flight requests finish on SIGTERM.

⚠️ Don't use `--reload` in production — it enables file watchers.

## Programmatic launch

```python
import uvicorn
if __name__ == "__main__":
    uvicorn.run("main:app", host="0.0.0.0", port=8000, workers=4)
```

## Gunicorn + Uvicorn workers

```bash
gunicorn main:app \
  -w 4 -k uvicorn.workers.UvicornWorker \
  --bind 0.0.0.0:8000 \
  --timeout 60 --graceful-timeout 30
```

Why bother: Gunicorn handles process supervision (restart on crash, preload), Uvicorn handles ASGI. See [[FastAPI 02 - Server Workers]].

## `fastapi run`

```bash
fastapi run main.py            # No reload, host 0.0.0.0, single process
fastapi run main.py --workers 4
```

Convenience wrapper around Uvicorn — fine for simple deploys.

## Process supervisor

Pick one:
- **systemd** unit on a VM.
- **Docker** restart policy ([[FastAPI 03 - Docker]]).
- **Kubernetes** Deployment + readiness/liveness probes.

⚠️ Crash without a supervisor = downtime. Always have one.

## Health check endpoint

```python
@app.get("/healthz", include_in_schema=False)
async def health(): return {"ok": True}
```

Wire to your load balancer / k8s probe.

💡 Containers + a process manager outside the container > running supervisord *inside* a container.
