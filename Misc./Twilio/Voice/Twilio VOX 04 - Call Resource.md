# VOX 04 — Call Resource

🔑 The `Call` resource (`CA…`) is the system of record for every leg — inbound, outbound, REST-created, dial-spawned. Same fields, same lifecycle.

Endpoint: `https://api.twilio.com/2010-04-01/Accounts/{AccountSid}/Calls[/{CallSid}].json`

## Fields

| Field | Notes |
|---|---|
| `sid` | `CA…` |
| `parent_call_sid` | If created by `<Dial>` from another call. |
| `account_sid` | Owner. |
| `to` / `from` | Both E.164 (or SIP / Client). |
| `to_formatted` / `from_formatted` | Human-friendly. |
| `phone_number_sid` | The `IncomingPhoneNumber` involved (inbound). |
| `status` | See lifecycle below. |
| `start_time` / `end_time` | RFC 2822. `null` until reached. |
| `duration` | Seconds, billable; `null` until completion. |
| `price` / `price_unit` | Negative number, e.g. `-0.0085`, `USD`. |
| `direction` | `inbound`, `outbound-api`, `outbound-dial`. |
| `answered_by` | If AMD was on: `human`, `machine_*`, `fax`, `unknown`. |
| `api_version` | `2010-04-01`. |
| `forwarded_from` | If forwarded by carrier (best-effort). |
| `caller_name` | CNAM lookup result. |
| `group_sid` | Internal grouping for parallel dials. |
| `queue_time` | Time spent in `<Enqueue>` queue. |
| `trunk_sid` | If via Elastic SIP Trunking. |

## Status lifecycle

```
queued ─→ initiated ─→ ringing ─→ in-progress ─→ completed
                                    │            ├─→ busy
                                    │            ├─→ no-answer
                                    │            ├─→ failed
                                    │            └─→ canceled
```

| Status | Meaning |
|---|---|
| `queued` | Outbound just created, dial not yet placed. |
| `initiated` | Dialing started. |
| `ringing` | Recipient ringback. |
| `in-progress` | Call answered; TwiML running. |
| `completed` | Hung up normally. |
| `busy` | Recipient was busy. |
| `no-answer` | Rang out without answer. |
| `failed` | Carrier-level failure. |
| `canceled` | Cancelled before connection (you `update(status="canceled")` on `queued` or `ringing`). |

Status callback events you can subscribe to: `initiated`, `ringing`, `answered`, `completed`. (See [[Foundations/Twilio 04 - Errors and Status Callbacks]].)

## CRUD

```python
# Create — see VOX 01.
call = client.calls.create(...)

# Fetch
call = client.calls("CA...").fetch()

# List
for c in client.calls.list(start_time_after=datetime(2026, 4, 1), status="completed", limit=200):
    ...

# Update — modify in flight or cancel
client.calls("CA...").update(twiml="<Response><Hangup/></Response>")
client.calls("CA...").update(status="canceled")    # only valid while queued/ringing
client.calls("CA...").update(status="completed")   # force hangup an in-progress call

# Delete (purge from logs after retention period; metadata retained)
client.calls("CA...").delete()
```

## Subresources

- **Recordings** — `client.calls("CA...").recordings.list()`. Each `RE…` is a recording. See [[Voice/Twilio VOX 05 - Recording]].
- **Notifications** — Twilio-side warnings/errors for the call (rate-limit, AMD events, …).
- **Events** — granular per-event log (paid feature: Voice Insights Advanced).
- **Payments** — captures from `<Pay>`.

## Modify-in-flight patterns

- **Live transfer** — `update(twiml="<Response><Dial>+1...</Dial></Response>")`.
- **Force hangup** — `update(status="completed")`.
- **Whisper before bridge** — `<Dial><Number url="/twiml/whisper">…</Number></Dial>` plays whisper TwiML to *callee* before bridging.

## Filtering tips

- `start_time` filter — UTC. Pass tz-aware `datetime`.
- `parent_call_sid` — pull child legs of a `<Dial>` (e.g., to find which agent answered).
- `direction="outbound-dial"` — calls Twilio created via `<Dial>` (vs your REST `outbound-api`).

## Pricing

- Voice billed per minute, rounded **up** to the next minute (or per-second on some plans).
- US local in/out ≈ USD 0.0085/min; international varies wildly — Lookup before dialing.
- Recording adds ~USD 0.0025/min storage + transcription (if enabled) ~USD 0.05/min.

⚠️ A `<Dial>` to a busy/no-answer destination is *not* free — you pay for the trying minute. Use Lookup for line-type checking before dialing toll-free / mobile-only numbers.
