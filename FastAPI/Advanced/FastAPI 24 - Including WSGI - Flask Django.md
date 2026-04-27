# Adv 24 — Including WSGI (Flask / Django)

🔑 Mount a legacy WSGI app inside FastAPI via `WSGIMiddleware`. New routes get FastAPI's full power; old routes keep working unchanged. Strangler-fig migration.

## Pattern

```python
from a2wsgi import WSGIMiddleware
from fastapi import FastAPI
from flask import Flask

flask_app = Flask(__name__)

@flask_app.route("/legacy")
def legacy():
    return "Hello from Flask"

app = FastAPI()

@app.get("/new")
async def new():
    return {"hello": "fastapi"}

app.mount("/v1", WSGIMiddleware(flask_app))     # /v1/legacy → flask
```

Run with Uvicorn — `a2wsgi` wraps the WSGI app to look like ASGI. (`fastapi.middleware.wsgi.WSGIMiddleware` is deprecated — install `a2wsgi` directly.)

## Django

```python
from django.core.wsgi import get_wsgi_application
import os

os.environ.setdefault("DJANGO_SETTINGS_MODULE", "myproj.settings")
django_app = get_wsgi_application()

app.mount("/admin", WSGIMiddleware(django_app))
```

## Trade-offs

- ✅ Reuse existing endpoints during a migration.
- ✅ Add async features, OpenAPI docs, dependency injection on new routes.
- ⚠️ WSGI calls run in a threadpool — slower than native ASGI under high concurrency.
- ⚠️ The WSGI app sees its own subset of the path (after `/v1` mount prefix).
- ⚠️ Middlewares apply *before* the WSGI mount; auth/logging in FastAPI middleware still runs.

## When to use

- Incremental migration from Flask/Django.
- Embedding admin tools (Django admin) inside an API.

## When not to

- Greenfield apps — just use FastAPI.
- High-throughput legacy endpoints — port them or keep two services behind a reverse proxy.

💡 Treat WSGI mounting as a temporary scaffold, not a permanent topology.
