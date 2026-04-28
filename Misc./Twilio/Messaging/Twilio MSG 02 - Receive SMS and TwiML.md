# MSG 02 — Receive SMS and Reply with TwiML

🔑 An inbound SMS triggers a webhook to your number's `sms_url`. You respond with **TwiML** (XML) — a short script Twilio executes against the *same conversation*. Return empty TwiML to ignore.

## Inbound webhook payload

POST (form-encoded) to `sms_url`, including:

| Field | Example |
|---|---|
| `MessageSid` | `SM…` |
| `AccountSid` | `AC…` |
| `From` | `+15551234567` (sender) |
| `To` | `+15557654321` (your Twilio number) |
| `Body` | text content |
| `NumMedia` | `0`+ |
| `MediaUrl0`, `MediaContentType0`, … | when MMS |
| `FromCity`, `FromState`, `FromCountry`, `FromZip` | best-effort geo |
| `MessagingServiceSid` | if attached to a service |

## TwiML reply — `<Message>`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Response>
  <Message>Thanks — we got it.</Message>
</Response>
```

Multiple `<Message>` verbs send multiple replies. Add `<Media>` for MMS:

```xml
<Response>
  <Message>
    Here is your receipt.
    <Media>https://app.example.com/r/1.pdf</Media>
  </Message>
</Response>
```

To stay silent, return an empty `<Response/>`.

## Build with the Python SDK

```python
from fastapi import APIRouter, Form
from fastapi.responses import Response
from twilio.twiml.messaging_response import MessagingResponse

router = APIRouter()

@router.post("/twilio/sms")
async def inbound_sms(From: str = Form(...), Body: str = Form(...)) -> Response:
    twiml = MessagingResponse()
    if Body.strip().upper() == "PING":
        twiml.message("PONG")
    return Response(content=str(twiml), media_type="application/xml")
```

⚠️ Always return `Content-Type: application/xml` (or `text/xml`). FastAPI's default JSON renderer breaks Twilio's parser.

## Async send instead of TwiML

If your reply takes more than a few seconds (LLM call, DB query), don't block the webhook — return empty TwiML, then send a follow-up via the REST API:

```python
@router.post("/twilio/sms", dependencies=[Depends(verify_twilio)])
async def inbound_sms(background: BackgroundTasks, From: str = Form(...), Body: str = Form(...)):
    background.add_task(reply_async, to=From, body=Body)
    return Response(content="<Response/>", media_type="application/xml")
```

## Opt-out keywords

Twilio (and US carriers) intercept these keywords automatically and update the recipient's opt-out state — your webhook may **not** even fire for them on Messaging Services with Advanced Opt-Out:

- **STOP / STOPALL / UNSUBSCRIBE / CANCEL / END / QUIT** → opt out.
- **HELP / INFO** → must reply with help text (carrier requirement).
- **START / YES / UNSTOP** → opt back in.

If you try to send to an opted-out number, the API returns `21610`. There's no override — the recipient must opt in again themselves.

## Idempotency / dedup

Twilio retries inbound delivery on 5xx. Use `MessageSid` as the dedup key in your inbound store.

## Multi-segment inbound

Long inbound messages may arrive split. Twilio auto-reassembles before calling your webhook in most regions, so `Body` is the full text. ⚠️ Some carriers / regions deliver as separate webhooks — check `NumSegments` and reconcile by `(From, To, ~time)` if needed.
