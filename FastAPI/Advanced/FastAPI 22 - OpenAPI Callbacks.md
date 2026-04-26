# Adv 22 — OpenAPI Callbacks

🔑 Document HTTP requests *your* API will make to *the client's* server (e.g. webhooks-on-completion). Pure docs feature — FastAPI doesn't actually issue these requests for you.

## Use case

You build `/invoices/`. When an invoice is paid, you `POST` to a URL the customer provided. Clients need to know what shape that callback will take. Callbacks describe it in OpenAPI.

## Define a callback router

```python
from fastapi import APIRouter, FastAPI
from pydantic import BaseModel, HttpUrl

class Invoice(BaseModel):
    id: str
    customer: str
    total: float

class InvoiceEvent(BaseModel):
    description: str
    paid: bool

class InvoiceEventReceived(BaseModel):
    ok: bool

callback_router = APIRouter()

@callback_router.post("{$callback_url}/invoices/{$request.body.id}",
                       response_model=InvoiceEventReceived)
def invoice_notification(body: InvoiceEvent): ...
```

`{$callback_url}` and `{$request.body.id}` are OpenAPI runtime expressions — they reference values from the original request.

## Wire it into the path op

```python
app = FastAPI()

@app.post("/invoices/", callbacks=callback_router.routes)
def create_invoice(invoice: Invoice, callback_url: HttpUrl):
    # ... do work ...
    # later: httpx.post(f"{callback_url}/invoices/{invoice.id}", json={...})
    return {"ok": True}
```

## What you actually get

The Swagger UI for `POST /invoices/` shows a *Callbacks* section with the shape of the outgoing webhook — clients learn from your docs alone.

## Implementation reminder

⚠️ FastAPI does **not** make the call for you. Use `httpx.AsyncClient` (probably from a [[FastAPI 22 - Background Tasks]] or queue worker).

## Webhooks vs callbacks

- **Callbacks** — tied to a specific path operation (one-off, per-request URL).
- **Webhooks** ([[FastAPI 23 - OpenAPI Webhooks]]) — global, app-level event subscriptions.

💡 Use callbacks when the URL is supplied *in the request*. Use webhooks when subscriptions are configured out-of-band.
