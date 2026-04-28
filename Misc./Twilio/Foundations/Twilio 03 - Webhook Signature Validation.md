# 03 — Webhook Signature Validation

🔑 Twilio signs every webhook with HMAC-SHA1 over the URL + sorted POST params, keyed by your **Auth Token**. You verify the `X-Twilio-Signature` header before trusting the request. Skip this and anyone who knows your URL can spoof Twilio.

## How Twilio signs

1. Take the **full request URL** (scheme, host, path, port, query string).
2. For form-encoded POSTs: sort POST params **alphabetically by name**, append each `name + value` (no separators) to the URL.
3. HMAC-SHA1 the resulting string using your **primary Auth Token** as the key.
4. Base64-encode → put in the `X-Twilio-Signature` header.

For `application/json` bodies: Twilio appends a `bodySHA256` *query parameter* (hex SHA-256 of the raw body) to the URL — your validator hashes the URL only, then compares the body hash separately. This avoids whitespace-normalisation mismatches.

## Validate (Python)

```python
from twilio.request_validator import RequestValidator

validator = RequestValidator(os.environ["TWILIO_AUTH_TOKEN"])

def is_valid(request) -> bool:
    sig = request.headers.get("X-Twilio-Signature", "")
    url = str(request.url)                      # full https://… URL
    params = dict(request.form)                 # form fields, NOT JSON body
    return validator.validate(url, params, sig)
```

For JSON: pass an empty `params` dict and the validator picks up `bodySHA256` from the URL query.

## FastAPI dependency

```python
from fastapi import Depends, HTTPException, Request, status
from twilio.request_validator import RequestValidator

_validator = RequestValidator(settings.twilio_auth_token)

async def verify_twilio(request: Request) -> None:
    sig = request.headers.get("X-Twilio-Signature", "")
    form = await request.form()
    if not _validator.validate(str(request.url), dict(form), sig):
        raise HTTPException(status.HTTP_403_FORBIDDEN, "bad twilio signature")

@app.post("/sms", dependencies=[Depends(verify_twilio)])
async def sms_webhook(...): ...
```

## Common pitfalls

⚠️ **Wrong URL.** Twilio signs the URL it called. If you're behind a reverse proxy / load balancer, `request.url` may show the internal scheme or host. Reconstruct the URL from `X-Forwarded-Proto` + `X-Forwarded-Host` + path, or set FastAPI's `root_path` and use `request.url`.

⚠️ **Stripped query string.** TwiML callbacks often include query params (e.g. `?CallSid=…`). They are part of the signed URL — keep them.

⚠️ **Voice HTTPS quirk.** For HTTPS voice callbacks, Twilio drops the port and the `userinfo` (user:pass@) before signing. The validator handles this; just don't manually inject ports.

⚠️ **Auth Token rotation.** Switching the primary Auth Token *invalidates outstanding signed requests in flight*. Rotate during a quiet window or use the secondary-token grace period documented in Console.

⚠️ **Don't dual-validate.** If a load balancer rewrites the URL/body before it reaches you, validation will silently fail. Disable rewrites for webhook routes.

## TLS requirements

- HTTPS only (no plaintext webhooks).
- Self-signed certs are rejected. Use a real CA (Let's Encrypt is fine).
- Don't pin Twilio's outbound certs — they rotate without notice.

## Defence in depth

- Bind webhooks to a *path with a random suffix* per environment so the URL itself acts like a shared secret.
- Network-allowlist Twilio's egress ranges if your platform supports it (published on `twilio.com/docs/usage/security`).
- Idempotency: use `MessageSid` / `CallSid` + status as your dedupe key — Twilio retries on 5xx.
