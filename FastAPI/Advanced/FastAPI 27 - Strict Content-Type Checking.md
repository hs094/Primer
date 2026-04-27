# Adv 27 — Strict Content-Type Checking

🔑 Since FastAPI 0.132.0, JSON request bodies require a valid `Content-Type` header (e.g. `application/json`) **by default**. Requests without one are rejected before the body is parsed.

## Default (strict)

```python
from fastapi import FastAPI
from pydantic import BaseModel

class Item(BaseModel):
    name: str
    price: float

app = FastAPI()

@app.post("/items/")
async def create_item(item: Item):
    return item
```

A `POST /items/` with no `Content-Type` (or one that isn't `application/json`) is rejected — body never reaches the handler.

## Opt out — app-level flag

```python
app = FastAPI(strict_content_type=False)
```

With `strict_content_type=False`, missing/wrong `Content-Type` headers are tolerated and the body is parsed as JSON anyway — the pre-0.132 behavior.

## Why strict by default

Defense against a narrow CSRF class: a browser script can issue a cross-origin request without a `Content-Type` header (skipping the CORS preflight). If the server happily parses it as JSON, an unauthenticated localhost/intranet app may execute attacker-controlled mutations. Requiring `application/json` forces the preflight, which CORS can then block.

- ⚠️ Mostly relevant for **localhost / internal-network** apps with no auth.
- For internet-exposed APIs with proper auth, the risk is already mitigated.

## When you might disable it

- Legacy clients that don't set `Content-Type`.
- Webhook senders / IoT devices with hardcoded behavior.
- Migrating a pre-0.132 app and need a temporary compat shim.

Otherwise — leave it on. It costs nothing and closes a real (if narrow) hole.

## Form / multipart routes

Unaffected. `Form()` / `File()` already require `multipart/form-data` or `application/x-www-form-urlencoded` and reject otherwise — no separate strict flag needed.

💡 New apps on FastAPI ≥ 0.132 — keep the default. Don't pass `strict_content_type=False` unless you've measured a concrete client breakage.
