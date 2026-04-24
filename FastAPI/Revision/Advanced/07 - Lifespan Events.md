# Adv 07 — Lifespan Events (startup / shutdown)

🔑 Use the **lifespan** async context manager to manage resources that span the app's lifetime.

## Modern `lifespan`

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI

@asynccontextmanager
async def lifespan(app: FastAPI):
    # ---- startup ----
    app.state.db = await create_pool(settings.database_url)
    app.state.ml_model = load_model(settings.model_path)
    yield
    # ---- shutdown ----
    await app.state.db.close()

app = FastAPI(lifespan=lifespan)
```

- Code before `yield` runs on startup.
- Code after `yield` runs on shutdown.
- Only one lifespan per app.

## Accessing state in handlers

```python
from fastapi import Request

@app.get("/predict")
async def predict(request: Request):
    model = request.app.state.ml_model
    return {"label": model.predict(...)}
```

Or inject a dependency that reads from `app.state`.

## Deprecated: `@app.on_event("startup") / "shutdown"`

Still works, but `lifespan` is the recommended replacement (cleaner structure, proper async resource semantics).

## Use cases

- DB connection pool, Redis client, HTTP client (`httpx.AsyncClient`).
- Load ML models, tokenizers.
- Warm caches, register with service discovery.
- Spin up background tasks; cancel them in shutdown.

## Lifespan with sub-apps

```python
app = FastAPI(lifespan=lifespan)
sub = FastAPI()                    # sub has its own lifespan
app.mount("/sub", sub)
```

- Mounted Starlette sub-apps run their own lifespan.
- To share resources: put state on the root `app.state` and pass a reference.

## Testing

```python
with TestClient(app) as client:
    ...     # lifespan fires inside `with`
```

## Gotchas

- ⚠️ Long blocking work in `lifespan` startup delays readiness checks. Keep it fast; background-initialize if needed.
- ⚠️ Exceptions during startup crash the server; log clearly.
- ⚠️ Don't mix `lifespan=` with `@app.on_event(...)` — pick one.
