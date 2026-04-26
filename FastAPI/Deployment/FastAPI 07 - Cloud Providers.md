# Deploy 07 — Deploy on Cloud Providers

🔑 Pick a deployment shape that matches the runtime model: container PaaS, serverless, VM, or k8s. Each has consequences for cold starts, lifespan events, and concurrency.

## Container PaaS (easiest)

- **Fly.io / Render / Railway / DigitalOcean App Platform**.
- Push a `Dockerfile` (or buildpack) → they run it. HTTPS, autoscale, logs included.
- Lifespan events run normally. Good for stateful connections (DB pools).

```dockerfile
FROM python:3.13-slim
WORKDIR /app
COPY pyproject.toml uv.lock ./
RUN pip install uv && uv sync --frozen
COPY . .
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

See [[FastAPI 03 - Docker]] for the full image pattern.

## Serverless containers

- **AWS App Runner / Google Cloud Run / Azure Container Apps**.
- Same Docker image. Scales to zero — cold starts hurt.
- ⚠️ Lifespan only fires on container start; expect occasional re-init costs.
- ⚠️ Don't expect long-lived background tasks — instances may suspend between requests.

## Serverless functions

- **AWS Lambda + Mangum**, **Vercel Functions**, **Google Cloud Functions Gen 2**.
- ASGI app wrapped in a handler.
- ⚠️ Long-poll / streaming / WebSockets are awkward or unsupported.
- ⚠️ Each cold start = lifespan boot. Heavy startups (ML model load) are punishing.

```python
# AWS Lambda
from mangum import Mangum
from main import app
handler = Mangum(app)
```

## Kubernetes

- Most control, most ops overhead.
- Deployment + Service + Ingress (with TLS via cert-manager).
- Configure liveness (`/healthz`), readiness (DB ready), and graceful shutdown (`terminationGracePeriodSeconds: 30`).

## VM (the old way)

- systemd unit running Gunicorn+UvicornWorker, Nginx in front.
- Cheap & predictable; ops you carry yourself.

## Cross-cutting checklist

- [ ] HTTPS terminated upstream — set `--proxy-headers` ([[FastAPI 06 - Run a Server Manually]]).
- [ ] `Content-Encoding: gzip` either upstream or via `GZipMiddleware`.
- [ ] Secrets via env vars / secret manager — never baked into image.
- [ ] Logs to stdout (12-factor); collector picks them up.
- [ ] Health/readiness endpoints wired to the platform.
- [ ] Autoscale on CPU + p99 latency, not RPS alone.

## Pick the simplest thing that fits

- One service, low traffic → PaaS container.
- Spiky / scale-to-zero ok → Cloud Run / Lambda.
- Multi-service company → k8s if a platform team exists.
- Hobby / single VM → systemd + Nginx.

💡 Don't choose k8s for "future proof" if today is one service.
