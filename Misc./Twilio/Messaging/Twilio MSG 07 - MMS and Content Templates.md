# MSG 07 — MMS and Content Templates

🔑 MMS = SMS with media (`MediaUrl`). The **Content API** is a higher-level abstraction: define a structured, reusable message once (text + buttons + media), send across SMS / MMS / WhatsApp / RCS by `ContentSid`.

## MMS basics

US/CA only on most numbers. Up to **10 attachments**, **5 MB total**.

```python
client.messages.create(
    to="+15551234567",
    from_="+15557654321",
    body="Order confirmed",
    media_url=[
        "https://app.example.com/img/order.jpg",
        "https://app.example.com/pdf/receipt.pdf",
    ],
)
```

Twilio fetches each URL via HTTP(S) at send time. URLs must be:

- **Publicly reachable** (no auth, no IP allowlist) for the duration of the send.
- Returning correct `Content-Type` (Twilio sniffs but trusts the header).
- Stable for at least the time it takes Twilio to fetch.

⚠️ Pre-signed S3 URLs work — make sure the expiry > a few minutes.

### Supported types (common)

`image/jpeg`, `image/png`, `image/gif`, `image/heic`, `audio/mpeg`, `audio/wav`, `video/mp4`, `application/pdf`, `text/vcard`. Carriers may downconvert (e.g., 4K image → resized).

### Inbound MMS

Webhook payload includes `NumMedia` and `MediaUrl0..N` + `MediaContentType0..N`. Fetch the media:

```python
import httpx, os
auth = (os.environ["TWILIO_ACCOUNT_SID"], os.environ["TWILIO_AUTH_TOKEN"])

async with httpx.AsyncClient(auth=auth, follow_redirects=True) as c:
    resp = await c.get(media_url0)
    blob = resp.content    # save / process
```

⚠️ Media URLs are auth-protected; you need basic auth with your account creds. They redirect to a CDN URL with an expiring signature.

## Content API (HX templates)

A Content template is a versioned, channel-agnostic message you author once and reference by `ContentSid` (`HX…`). Critical for WhatsApp templates and quoted-reply / button experiences on RCS, but works for plain SMS too.

### Content types

| Type | Channels | Use |
|---|---|---|
| `twilio/text` | SMS, WhatsApp, RCS | plain text with `{{variables}}` |
| `twilio/media` | MMS, WhatsApp, RCS | text + media |
| `twilio/quick-reply` | WhatsApp, RCS | tap-button replies (max 10) |
| `twilio/call-to-action` | WhatsApp, RCS | URL / phone-number buttons |
| `twilio/list-picker` | WhatsApp, RCS | pick from a list |
| `twilio/card` | RCS, WhatsApp | image+title+description+buttons |
| `twilio/carousel` | RCS | swipeable cards |
| `twilio/location` | WhatsApp | geo coordinates |
| `whatsapp/authentication` | WhatsApp | one-time code with copy button |

### Create a template

```python
template = client.content.v2.contents.create(
    friendly_name="order_shipped_v1",
    language="en",
    variables={"1": "name", "2": "tracking_url"},
    types={
        "twilio/text": {"body": "Hi {{1}} — your order has shipped. Track at {{2}}"},
    },
)
print(template.sid)   # HX...
```

For WhatsApp, submit for Meta approval before sending:

```python
client.content.v1.contents(template.sid).approval_create(
    name="order_shipped_v1",
    category="UTILITY",   # or AUTHENTICATION / MARKETING
)
```

### Send via template

```python
client.messages.create(
    to="whatsapp:+15551234567",
    from_="whatsapp:+14155238886",
    content_sid="HX...",
    content_variables='{"1":"Pranav","2":"https://x.com/t/abcd"}',
)
```

`content_variables` is a JSON-string (note: string, not dict) of `{"1": "...", "2": "..."}` matching declared variables.

### Cross-channel sends

The same `ContentSid` may render differently on SMS vs WhatsApp vs RCS — Twilio picks the matching `types/*` definition. If a channel doesn't have a matching type, send fails with a content-mismatch error.

## When to prefer Content over `body`

- WhatsApp templates (required for out-of-window sends).
- RCS-rich experiences with buttons / cards.
- Anything you'll send 100s of times — easier to A/B-test or update.
- Compliance: regulators (and Meta) look for *templates*, not freeform.
