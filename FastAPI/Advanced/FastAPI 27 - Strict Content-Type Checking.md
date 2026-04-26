# Adv 27 — Strict Content-Type Checking

🔑 By default FastAPI tolerates a missing or wrong `Content-Type` for JSON bodies. Strict mode rejects bodies whose declared content type doesn't match what the route expects.

## Default (lenient)

A `POST /items/` with `Content-Type: text/plain` and a JSON body might still be parsed — surprising but historic behavior.

## Opt into strict mode

```python
from fastapi import Body

@app.post("/items/")
async def create(item: Item = Body(..., media_type="application/json", strict=True)):
    ...
```

(Or per-app via configuration where supported.) When `strict=True`, requests without `application/json` (or matching `media_type`) get a 415.

## Why strict

- Defense against clients sending malformed/poisoned payloads.
- Makes API contracts honest in the docs.
- Avoids accidental success with wrong content (e.g. proxy stripped the header).

## Explicit content negotiation

```python
@app.post("/items/", openapi_extra={
    "requestBody": {"content": {"application/json": {"schema": {...}}}}
})
```

Pair strict checking with explicit `responses=` declarations so docs and runtime agree.

## Multipart / form

For form endpoints, use `Form()` / `File()` — those already require `multipart/form-data` or `application/x-www-form-urlencoded`. No strict opt-in needed.

## Custom rejection

If you want a richer error than 415:

```python
from fastapi import Request, HTTPException

@app.middleware("http")
async def enforce_json(request: Request, call_next):
    if request.method in {"POST", "PUT", "PATCH"} and request.url.path.startswith("/api/"):
        ct = request.headers.get("content-type", "")
        if not ct.startswith("application/json"):
            raise HTTPException(415, "must be application/json")
    return await call_next(request)
```

⚠️ Middleware-level checks are blunt — prefer per-route `Body(strict=True)` so contracts stay co-located.

💡 Lenient defaults exist for backward compat. New APIs — turn strict on from day one.
