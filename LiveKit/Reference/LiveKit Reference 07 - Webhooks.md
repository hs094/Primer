# 07 — Webhooks

🔑 Webhooks push room/participant/egress/ingress lifecycle events to your backend, signed as a JWT containing a SHA256 of the body — verify before trusting.

Source: https://docs.livekit.io/home/server/webhooks/

## Event types

- `room_started`, `room_finished`
- `participant_joined`, `participant_left`
- `track_published`, `track_unpublished`
- `egress_started`, `egress_updated`, `egress_ended`
- `ingress_started`, `ingress_ended`

`participant_joined` fires only after the media connection is established and state is `ACTIVE` — not at signal connect.

## Configure (Cloud)

Dashboard → Settings → Webhooks → add endpoint URL + signing API key.

## Verify with Python SDK

```python
from fastapi import FastAPI, Request, HTTPException
from livekit import api

app = FastAPI()
token_verifier = api.TokenVerifier()  # uses LIVEKIT_API_KEY/SECRET
webhook_receiver = api.WebhookReceiver(token_verifier)

@app.post("/livekit/webhook")
async def webhook(req: Request) -> dict[str, str]:
    body = (await req.body()).decode("utf-8")
    auth = req.headers.get("Authorization")
    if not auth:
        raise HTTPException(401, "missing auth")

    event = webhook_receiver.receive(body, auth)  # raises on bad signature

    match event.event:
        case "room_started":
            print(f"room {event.room.name} started")
        case "participant_joined":
            print(f"{event.participant.identity} joined {event.room.name}")
        case "egress_ended":
            print(f"egress {event.egress_info.egress_id} done")

    return {"ok": "1"}
```

## Delivery semantics

- No delivery guarantees — push-based, may drop on transient failures.
- LiveKit retries with backoff on non-2xx.
- Order is preserved per event sequence but not strictly across topics — make handlers idempotent (event has a unique `id`).

## Auth header

`Authorization: <jwt>` where the JWT body's `sha256` claim equals `base64(sha256(raw_body))`. SDK's `WebhookReceiver` validates both signature and payload hash.

## Tags
[[LiveKit]] [[Reference]] [[Webhooks]] [[Events]]
