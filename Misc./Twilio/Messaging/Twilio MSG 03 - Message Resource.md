# MSG 03 — Message Resource

🔑 The `Message` resource is the system of record for *every* SMS / MMS / WhatsApp / RCS message. Same shape, regardless of channel.

Endpoint: `https://api.twilio.com/2010-04-01/Accounts/{AccountSid}/Messages[/{MessageSid}].json`

## Fields

| Field | Type | Notes |
|---|---|---|
| `sid` | str | `SM…` (SMS) or `MM…` (MMS). |
| `account_sid` | str | Owner. |
| `messaging_service_sid` | str \| null | If sent through a service. |
| `from` | str | Sender (E.164 or `whatsapp:+…` or alphanumeric). |
| `to` | str | Recipient. |
| `body` | str | Up to 1600 chars (multi-segment). |
| `num_segments` | str | Count of SMS segments. |
| `num_media` | str | Count of media items. |
| `direction` | enum | `inbound`, `outbound-api`, `outbound-call`, `outbound-reply`. |
| `status` | enum | See below. |
| `error_code` / `error_message` | int / str | Set on failure. |
| `price` / `price_unit` | str | Final billed price (negative number; `USD`). |
| `date_created` / `date_sent` / `date_updated` | datetime (RFC 2822) | Lifecycle timestamps. |
| `uri` | str | Self-link. |
| `subresource_uris` | dict | `media`, `feedback`. |

## Status lifecycle

```
              ┌─→ accepted ─→ scheduled ─→ queued ─→ sending ─→ sent ─→ delivered
outbound-api ─┤                                                   │      │
              │                                                   │      ├─→ read       (RCS / WhatsApp)
              │                                                   ├─→ undelivered
              │                                                   └─→ failed
inbound ─→ receiving ─→ received
                                       canceled  (scheduled message canceled before send)
```

| Status | Meaning |
|---|---|
| `accepted` | Messaging Service has the message; sender not yet picked. |
| `scheduled` | Held for `SendAt` time. |
| `queued` | Twilio has it, awaiting carrier hand-off. |
| `sending` | Handed to carrier. |
| `sent` | Carrier accepted. (No delivery confirmation yet.) |
| `failed` | Couldn't be sent (auth, formatting, carrier reject). |
| `delivered` | Carrier confirmed handset delivery. |
| `undelivered` | Carrier returned a delivery failure. |
| `read` | Recipient opened (RCS / WhatsApp only). |
| `received` | Inbound message stored. |
| `receiving` | Inbound in flight (rare to observe). |
| `canceled` | Scheduled message cancelled. |
| `partially_delivered` | Group send (rare; mostly RCS). |

## CRUD via Python

```python
# Create — see MSG 01.
msg = client.messages.create(...)

# Fetch a specific message
msg = client.messages("SM...").fetch()

# List (filter by date / to / from)
for m in client.messages.list(date_sent_after=datetime(2026, 4, 1), limit=200):
    print(m.sid, m.status, m.to)

# Redact body / media after the fact (privacy)
client.messages("SM...").update(body="")

# Cancel a scheduled message
client.messages("SM...").update(status="canceled")

# Delete (purges from logs)
client.messages("SM...").delete()
```

## Media subresource

```python
media_list = client.messages("MM...").media.list()
for m in media_list:
    print(m.sid, m.content_type, m.uri)   # MEdiaSid + relative URI
```

Each `MediaInstance` has a `content_type` and a `uri`. Fetch the binary by hitting `https://api.twilio.com{uri}` (the URI redirects to a CDN URL).

## Pagination

`messages.list(...)` auto-paginates. For large pulls use `messages.stream(...)` (generator) or pass `page_size=` and iterate manually to avoid loading every page in memory.

## Filters worth knowing

```python
client.messages.list(
    to="+15551234567",
    from_="+15557654321",
    date_sent_after=...,
    date_sent_before=...,
)
```

⚠️ `date_sent` filter uses **UTC**. Pass `datetime` aware of timezone.

## Retention

Default body retention is per Messaging Privacy settings (Console). After the retention window, `body` and `media` return empty / 404 even though `sid` and metadata persist for billing.
