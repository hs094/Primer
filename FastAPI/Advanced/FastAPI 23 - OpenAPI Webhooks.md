# Adv 23 — OpenAPI Webhooks

🔑 Declare events your API *emits* to subscribers. Like callbacks, but global and not bound to one request. OpenAPI 3.1+ feature.

## Declare a webhook

```python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()

class Subscription(BaseModel):
    username: str
    new_plan: str

@app.webhooks.post("new-subscription")
def new_subscription(body: Subscription):
    """When a new user subscribes, you'll receive a POST with the details."""
```

The path string is the **event name**, not a URL. Subscribers configure their own receiver URL elsewhere (admin panel, env var, etc.).

## Result in docs

Swagger renders a "Webhooks" section listing every event with its request schema and (optional) response.

## Multiple events

```python
@app.webhooks.post("user.created")
def user_created(body: UserCreated): ...

@app.webhooks.post("order.shipped")
def order_shipped(body: OrderShipped): ...
```

## Triggering

⚠️ FastAPI does **not** dispatch webhooks for you. Maintain a subscriber registry, fan out via `httpx.AsyncClient` (likely from a queue worker).

## Vs callbacks

| | Callbacks | Webhooks |
|---|---|---|
| URL source | Per-request body | Out-of-band subscription |
| Scope | One path op | Whole app |
| OpenAPI | `callbacks=` on op | `app.webhooks.post(...)` |

## Versioning the events

Treat event payloads as a public contract. Add fields, never remove. Breaking change → new event name (e.g. `order.shipped.v2`).

💡 Don't forget retries, idempotency keys, and signing on the *delivery* side — OpenAPI only documents the shape.
