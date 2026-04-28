# MSG 05 — WhatsApp

🔑 Same `Messages` API, prefix addresses with `whatsapp:`. Two big rules: (1) you must use **approved templates** to start a conversation; (2) freeform replies are only allowed inside the **24-hour customer service window** after the user messages you.

## Numbers and senders

- Production: a phone number **registered to a WhatsApp Business Account (WABA)**. Get there via:
  - **WhatsApp Self Sign-up** (direct customers), or
  - **Tech Provider Program** (ISVs / platforms onboarding many merchants).
- Dev: the **Twilio Sandbox** (`whatsapp:+14155238886`). Users join by texting a join code; great for prototyping, *not* for production.

## Sandbox quickstart

1. In Console → Messaging → Try it out → Send a WhatsApp message — note the join code.
2. From your phone: `WhatsApp` → `+1 415 523 8886` → send `join <code>`.
3. Now you can SDK-send to that number for ~72 hours.

## Send a freeform message (inside 24h window)

```python
client.messages.create(
    from_="whatsapp:+14155238886",
    to="whatsapp:+15551234567",
    body="Sure — I'll mark you down for Friday.",
)
```

⚠️ Outside the 24h window, freeform sends fail. Use a template (Content) instead.

## Send a template (any time)

Templates are pre-approved by Meta (subject, content, variables). Twilio represents them as **Content templates** with a `ContentSid` (`HX…`).

```python
client.messages.create(
    from_="whatsapp:+14155238886",
    to="whatsapp:+15551234567",
    content_sid="HX...",
    content_variables='{"1":"Pranav","2":"Friday 4pm"}',  # JSON string
)
```

Categories Meta approves:

| Category | Use |
|---|---|
| **Authentication** | OTPs, login codes |
| **Utility** | order updates, alerts about something the user did |
| **Marketing** | promotional sends — most expensive, opt-in heavy |

## Inbound webhook

Same shape as SMS inbound (see [[Messaging/Twilio MSG 02 - Receive SMS and TwiML]]) — `From` and `To` are prefixed `whatsapp:+…`. Reply with TwiML `<Message>` or via REST.

## Media

Send images, audio, video, PDF, stickers, contact cards, locations:

```python
client.messages.create(
    from_="whatsapp:+14155238886",
    to="whatsapp:+15551234567",
    body="Receipt attached",
    media_url=["https://app.example.com/r/1.pdf"],
)
```

## 24-hour window: practical rules

- Window starts at **each inbound user message** and resets on the next inbound.
- Freeform (any `Body`/`Media` you author) → only in window.
- Template (any `ContentSid`) → any time. Required to "open" a new conversation.
- Mixing is fine: send a template to start, then freeform within 24h.

## Opt-in

Required by Meta. You must collect explicit opt-in (web checkbox, IVR, prior chat) *before* sending a template. Twilio doesn't enforce this for you, but Meta will throttle / suspend WABAs that violate policy.

## Pricing model

Conversation-based, not per-message. Each opened conversation (initiated by template or by user) costs once per 24-hour window, plus the per-message Twilio fee. Categories price differently (Authentication < Utility < Marketing).

## Common errors

| Code | Meaning |
|---|---|
| 63016 | Failed to send freeform message; outside 24h window — use a template. |
| 63003 | Channel could not find recipient (number not on WhatsApp). |
| 63007 | Channel `From` address invalid — sender not registered. |
| 63018 | Rate limit hit — Meta-side throttle. |
| 63032 | Recipient blocked the sender. |

💡 If you're seeing a lot of `63016`, your business logic is treating templates as freeform. Audit which sends go via `ContentSid` vs `body`.
