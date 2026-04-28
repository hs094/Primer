# MSG 04 — Messaging Services

🔑 A **Messaging Service** (`MG…`) is a sender pool plus routing logic plus a single inbound URL. Use one once you have more than one number, more than one country, or any 10DLC sending. Single number? You can skip it.

## What a service gives you

- **Sender Pool** — attach multiple numbers (long codes, toll-free, short codes), alphanumeric IDs, RCS/WhatsApp senders.
- **Sticky Sender** — recipient X always gets messages from the same `From` number.
- **Country Code Geomatch** — sender country auto-matches recipient. US fallback if no local sender available.
- **Area Code Geomatch** — for US/CA, prefer a sender with the recipient's area code.
- **Smart Encoding** — silently swap lookalike Unicode (`’` → `'`) to keep messages in GSM-7 (cheap).
- **MMS Converter** — fall back to SMS+shortened URL when MMS unsupported.
- **Link Shortening** + **Click Tracking** — service-level toggle.
- **Scheduled Messages** — `SendAt` + `ScheduleType=fixed`, up to 7 days ahead.
- **Validity Period** — extend up to 36000s (vs. 14400s on raw send).
- **Advanced Opt-Out** — custom STOP/HELP/START keywords + replies, per language.
- **Inbound Routing** — one webhook URL covers every number in the pool.
- **A2P 10DLC** — service is the unit you attach a Brand + Campaign to (see [[Messaging/Twilio MSG 06 - 10DLC and Compliance]]).

Sender priority when multiple match: **RCS → short codes → alphanumeric IDs → long codes / toll-free**.

## Create a service

```python
service = client.messaging.v1.services.create(
    friendly_name="Marketing US",
    inbound_request_url="https://app.example.com/twilio/sms",
    fallback_url="https://app.example.com/twilio/sms-fallback",
    status_callback="https://app.example.com/twilio/sms-status",
    sticky_sender=True,
    smart_encoding=True,
    area_code_geomatch=True,
    use_inbound_webhook_on_number=False,   # let the service own inbound
)

# Attach numbers
client.messaging.v1.services(service.sid).phone_numbers.create(phone_number_sid="PN...")
```

## Send via service

```python
client.messages.create(
    to="+15551234567",
    messaging_service_sid=service.sid,
    body="hello",
)
```

No `From` — service picks. The chosen sender appears on the resulting `Message` resource as `from`.

## Inbound routing options

| `use_inbound_webhook_on_number` | Behaviour |
|---|---|
| `False` (default) | Service's `inbound_request_url` handles all messages. |
| `True` | Each number's own `sms_url` handles its own inbound (legacy / per-number routing). |

`Auto-create Conversations` is a third option — every inbound becomes a `Conversation` resource (see [[Conversations/Twilio CV 01 - Conversations]]).

## Scheduling

```python
from datetime import datetime, timedelta, timezone

client.messages.create(
    to="+15551234567",
    messaging_service_sid="MG...",
    body="reminder",
    send_at=datetime.now(timezone.utc) + timedelta(hours=4),
    schedule_type="fixed",
)
```

⚠️ Scheduling requires a Messaging Service. `send_at` must be ≥ 15 minutes future and ≤ 7 days.

## Cancellation

```python
client.messages("SM...").update(status="canceled")    # only valid while scheduled
```

## Throughput notes

- Pool throughput = sum of per-sender MPS, capped per-channel by carrier rules.
- 10DLC long codes: throughput depends on **Trust Score** (1–10) of your registered Brand.
- Toll-free: ~3 MPS per number once verified.
- Short codes: 100 MPS typical.
- Alphanumeric IDs: country-dependent; US doesn't allow them inbound.

## Advanced Opt-Out

Customise the auto-replies for STOP / HELP / START per language and country in Console → Messaging → Opt-Out Management. Carriers still enforce STOP regardless of your settings.

💡 If you don't configure HELP, US carriers may filter your campaign.
