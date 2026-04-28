# 02 — Helper Libraries and Auth

🔑 The helper library is a thin REST wrapper plus a TwiML builder plus a request validator. You can do everything with `httpx` and string-formatting; the library just saves typing.

## Languages

Official server SDKs: **Python**, **Node.js**, **PHP**, **Java**, **C# / .NET**, **Ruby**, **Go**.
Client-side SDKs: **JavaScript**, **iOS**, **Android**, **React Native** (Voice, Video, Conversations, Verify push, …).

## Install (Python)

```bash
uv add twilio          # preferred
# or:
pip install twilio
```

## Minimal client

```python
import os
from twilio.rest import Client

client = Client(
    os.environ["TWILIO_ACCOUNT_SID"],
    os.environ["TWILIO_AUTH_TOKEN"],
)
```

`Client(...)` is sync. There is also `twilio.rest.Client` with HTTPX support for async helpers — but for a FastAPI service the common pattern is to call from a threadpool (`run_in_threadpool` / `to_thread.run_sync`) since the SDK is sync-first.

## Auth options

| Method | What you sign with | Use when |
|---|---|---|
| **AccountSid + AuthToken** | Primary auth token | Local dev, single-account scripts |
| **API Key (Standard)** | `SK…` SID + secret | Server apps — rotatable, revocable |
| **API Key (Master)** | `SK…` SID + secret with elevated scope | Account-level admin operations |
| **OAuth (Twilio Connect)** | Third-party access token | Apps acting on behalf of *another* Twilio account |
| **Public Key Client Validation** | Asymmetric key signing | High-security / no-shared-secret deployments |

```python
# API key auth — preferred for server apps
client = Client(
    os.environ["TWILIO_API_KEY_SID"],       # SK...
    os.environ["TWILIO_API_KEY_SECRET"],
    os.environ["TWILIO_ACCOUNT_SID"],       # account context
)
```

⚠️ AuthToken doubles as the **HMAC key for webhook signatures**. Rotating the primary token rotates webhook validation — coordinate with your handlers.

## Resource access pattern

Every product hangs off `client.<product>`:

```python
client.messages.create(...)            # SMS / MMS / WhatsApp
client.calls.create(...)               # Voice
client.verify.v2.services(sid).verifications.create(...)
client.lookups.v2.phone_numbers(num).fetch(fields="line_type_intelligence")
client.conversations.v1.conversations.create(...)
client.video.v1.rooms.create(...)
client.studio.v2.flows(sid).executions.create(...)
client.serverless.v1.services(sid).environments(eid).deployments.create(...)
```

💡 Mental model: `client.<product>.v<n>.<resource>(sid).<sub>(...).<verb>(...)`.

## TwiML builder

```python
from twilio.twiml.voice_response import VoiceResponse

resp = VoiceResponse()
resp.say("Hello from Twilio.", voice="Polly.Joanna")
resp.hangup()
print(str(resp))   # XML string
```

`MessagingResponse`, `VoiceResponse`, `VideoResponse` mirror the TwiML verbs. Output is the XML you return from a webhook.

## Storing credentials

- **`pydantic-settings`** with a `Settings(BaseSettings)` class — never raw `os.environ.get(...)` scattered across modules.
- Never log the auth token. Never commit it. Rotate via Console; the old token keeps working briefly to allow drainage.
- For multi-tenant apps, store the *customer's* AccountSid + API Key, not auth token.
