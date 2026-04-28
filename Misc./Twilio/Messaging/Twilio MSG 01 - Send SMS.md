# MSG 01 — Send SMS

🔑 `POST /2010-04-01/Accounts/{Sid}/Messages.json` with `To`, `From` (or `MessagingServiceSid`), and one of `Body` / `MediaUrl` / `ContentSid`. That's the entire send API.

## Minimal send (Python)

```python
msg = client.messages.create(
    to="+15551234567",
    from_="+15557654321",
    body="Your code is 123456.",
)
print(msg.sid, msg.status)   # SM..., queued
```

`status` returned synchronously is almost always `queued` — actual delivery shows up later via status callback (see [[Foundations/Twilio 04 - Errors and Status Callbacks]]).

## Required parameters

| Param | Required when |
|---|---|
| `To` | Always. E.164 (`+1…`). For WhatsApp prefix `whatsapp:+…`. |
| `From` *or* `MessagingServiceSid` | Pick one. Service is preferred at scale. |
| `Body` *or* `MediaUrl` *or* `ContentSid` | Need at least one. |

## Common optional parameters

| Param | Purpose |
|---|---|
| `StatusCallback` | Webhook for delivery state transitions. |
| `MaxPrice` | Hard cap on price-per-message. Twilio refuses to send above. |
| `ProvideFeedback` | If true, you intend to call `MessageFeedback` later (carrier delivery confirm). |
| `ValidityPeriod` | Drop the message if not sent within N seconds (1–14400 raw, up to 36000 via service). |
| `SendAt` + `ScheduleType=fixed` | Schedule in the future (Messaging Service required). |
| `SmartEncoded` | Replace lookalike Unicode with GSM equivalents to keep segments cheap. |
| `ShortenUrls` | Auto-shorten links (Messaging Service). |
| `MessagingServiceSid` | Attach to service — enables Sticky Sender, 10DLC routing, etc. |

## Multiple media

```python
client.messages.create(
    to="+15551234567",
    from_="+15557654321",
    body="here are the receipts",
    media_url=[
        "https://app.example.com/r/1.jpg",
        "https://app.example.com/r/2.pdf",
    ],
)
```

Up to 10 media URLs per MMS. Twilio fetches each via HTTP(S); they must be public for the duration of the send. Supported types include common image (jpg/png/gif), audio, video, and PDF — full list per carrier.

## Segments & encoding

- **GSM-7**: 160 chars per segment. 153 if multi-segment (header takes 7).
- **UCS-2** (any non-GSM char, e.g. emoji): 70 chars per segment. 67 if multi-segment.
- One segment = one billable message. Long messages are reassembled by the handset.
- 💡 A single `’` (curly apostrophe) flips the whole message to UCS-2. Use `SmartEncoded=true` or normalise text.

`num_segments` on the returned `Message` resource tells you the count.

## E.164 normalisation

- Always pass `+` + country + number, no spaces.
- Use `phonenumbers` (Python) or Twilio Lookup v2 to normalise user input *before* sending.

```python
import phonenumbers
e164 = phonenumbers.format_number(
    phonenumbers.parse(raw, "US"),
    phonenumbers.PhoneNumberFormat.E164,
)
```

## Send to WhatsApp (same endpoint)

```python
client.messages.create(
    to="whatsapp:+15551234567",
    from_="whatsapp:+14155238886",     # Twilio sandbox or your registered sender
    body="hello over WhatsApp",
)
```

See [[Messaging/Twilio MSG 05 - WhatsApp]] for templates / 24h session rules.

## Use a Messaging Service

```python
client.messages.create(
    to="+15551234567",
    messaging_service_sid="MG...",
    body="from the pool",
)
```

The service picks the `From` number per recipient (Sticky Sender, geomatch). Required for A2P 10DLC at scale. See [[Messaging/Twilio MSG 04 - Messaging Services]].
