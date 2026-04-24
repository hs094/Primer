# Dep 03 — Docker

🔑 Container ships a **single worker process** per container; the orchestrator scales replicas. One migration step as a separate job.

## Minimal Dockerfile

```dockerfile
FROM python:3.12-slim

WORKDIR /code

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY ./app /code/app

CMD ["fastapi", "run", "app/main.py", "--port", "80"]
```

## With `uv` (preferred)

```dockerfile
FROM python:3.12-slim

ENV UV_SYSTEM_PYTHON=1 \
    UV_COMPILE_BYTECODE=1 \
    UV_LINK_MODE=copy

COPY --from=ghcr.io/astral-sh/uv:latest /uv /usr/local/bin/uv

WORKDIR /code

COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev

COPY ./app /code/app

EXPOSE 80
CMD ["uv", "run", "fastapi", "run", "app/main.py", "--port", "80"]
```

## Multi-stage build

```dockerfile
# --- builder ---
FROM python:3.12-slim AS builder
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt

# --- runtime ---
FROM python:3.12-slim
COPY --from=builder /install /usr/local
COPY ./app /code/app
WORKDIR /code
CMD ["fastapi", "run", "app/main.py", "--port", "80"]
```

Smaller final image (no build tooling).

## One process per container?

Yes, typically. Let Kubernetes / ECS replicate.

Exceptions where multiple workers in one container are fine:
- Small VMs / single-host deploys without an orchestrator.
- When cold-start cost is high (ML models) and you want concurrency per instance.

If doing multiple workers: `fastapi run main.py --workers 4` or Gunicorn + Uvicorn workers.

## Healthchecks

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s \
  CMD python -c "import httpx,sys; sys.exit(0 if httpx.get('http://localhost/healthz').is_success else 1)"
```

Or let Kubernetes do it with `livenessProbe` / `readinessProbe`.

## Migrations

Run as a separate one-off before rolling out app containers:

```bash
docker run --rm myapp alembic upgrade head
```

In K8s: use a `Job` or `initContainer` on deploy.

## Env / secrets

- `.env` files: fine locally; inject at runtime (`docker run --env-file` or K8s `Secret`) in prod.
- Don't bake secrets into images.

## Image size tips

- Use `python:*-slim`; pin exact version.
- Combine RUN steps; `--no-cache-dir` for pip.
- `.dockerignore` — keep `.git`, `__pycache__`, `.venv`, tests out.

## Gotchas

- ⚠️ `CMD ["uvicorn", "main:app", "--reload"]` → never in prod images.
- ⚠️ `PYTHONUNBUFFERED=1` for stdout to flush to container logs.
- ⚠️ Bind 0.0.0.0 inside the container; your host port comes from `-p`.
