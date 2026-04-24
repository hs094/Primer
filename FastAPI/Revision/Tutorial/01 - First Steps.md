# 01 — First Steps

🔑 FastAPI = Starlette (ASGI) + Pydantic (validation) + OpenAPI (docs) with type hints as the single source of truth.

## Install

```bash
pip install "fastapi[standard]"   # includes uvicorn, httpx, jinja2, python-multipart, email-validator
pip install fastapi               # slim, bring-your-own server
```

`[standard]` pulls in the `fastapi` CLI (Uvicorn) and common extras. Slim install if you want to pin your own stack.

## Minimal app

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
async def root():
    return {"hello": "world"}
```

## Run

```bash
fastapi dev  main.py   # auto-reload, localhost only, DEV
fastapi run  main.py   # no reload, binds 0.0.0.0, PROD-ish
uvicorn main:app --reload --port 8000   # explicit
```

- `dev` ⇒ reload=True, host=127.0.0.1.
- `run` ⇒ reload=False, host=0.0.0.0. Use a real process manager in production.

## Auto docs (free)

- `/docs` → Swagger UI
- `/redoc` → ReDoc
- `/openapi.json` → raw OpenAPI schema

## Path operation decorators

`@app.get | post | put | delete | patch | options | head | trace`.

Each decorated function = **path operation function**. Return value is serialized to JSON by default.

## Async vs sync

- `async def` → runs on the event loop. Use for `await`-able I/O.
- `def` → runs in threadpool (AnyIO). FastAPI picks correctly; blocking sync is safe but slower.
- ⚠️ Don't mix blocking I/O inside `async def` (blocks the loop).

## Type hints drive everything

- Function params → request parsing (path / query / body / header / cookie).
- Return annotation + `response_model` → response schema + filtering.
- Pydantic models → validation + docs + editor autocomplete.

💡 If you can express a constraint in a type, prefer that over runtime checks.

## Interactive docs out of the box

Every path op, parameter, response, example, and description surface automatically. You rarely hand-write OpenAPI.
