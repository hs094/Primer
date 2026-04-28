# 04 — Errors and Status Callbacks

🔑 Two failure surfaces: (a) the **synchronous** REST call (HTTP 4xx/5xx with a Twilio error code), and (b) the **asynchronous** delivery outcome that arrives later via a status callback. You must handle both.

## REST errors

```python
from twilio.base.exceptions import TwilioRestException

try:
    msg = client.messages.create(to="+15551234567", from_="+15557654321", body="hi")
except TwilioRestException as e:
    e.status        # HTTP status, e.g. 400
    e.code          # Twilio error code, e.g. 21211
    e.msg           # human-readable
    e.more_info     # https://www.twilio.com/docs/errors/<code>
```

- HTTP `400` / `404` / `409` → permanent. Don't retry blindly. Look up the code.
- HTTP `429` → rate-limited. Back off. Twilio publishes per-product limits.
- HTTP `5xx` → retry with jitter. Idempotency key recommended for `messages.create` (see below).

## Frequently seen codes

| Code | Meaning |
|---|---|
| 20003 | Authentication failure (bad SID/token/key) |
| 20404 | Resource not found |
| 20429 | Too many requests |
| 21211 | Invalid `To` number (not E.164) |
| 21212 | Invalid `From` number |
| 21408 | Permission to send to that region not enabled (Geo Permissions) |
| 21610 | Recipient previously opted out (STOP) |
| 21611 | `From` number is not SMS-capable |
| 21614 | `To` is not a mobile number |
| 30003 | Unreachable destination handset |
| 30004 | Message blocked |
| 30005 | Unknown destination |
| 30006 | Landline / unreachable carrier |
| 30007 | Carrier filtering |
| 30008 | Unknown error from carrier |
| 30034 | A2P 10DLC not registered (US) |

Full reference: https://www.twilio.com/docs/api/errors

## Status callbacks (the async half)

Pass a `StatusCallback` URL when creating the resource. Twilio POSTs progress events to it.

### Messages

```python
client.messages.create(
    to="+15551234567",
    from_="+15557654321",
    body="hi",
    status_callback="https://app.example.com/twilio/sms-status",
)
```

Events delivered: `queued` → `sending` → `sent` → `delivered` (or `failed` / `undelivered`).
Body fields you'll receive: `MessageSid`, `MessageStatus`, `ErrorCode`, `ErrorMessage`, `From`, `To`, `AccountSid`.

### Calls

```python
client.calls.create(
    to="+15551234567",
    from_="+15557654321",
    url="https://app.example.com/twiml",
    status_callback="https://app.example.com/twilio/call-status",
    status_callback_event=["initiated", "ringing", "answered", "completed"],
    status_callback_method="POST",
)
```

Events: `initiated`, `ringing`, `answered`, `completed`. Body includes `CallSid`, `CallStatus`, `CallDuration`, `Direction`, `From`, `To`, `Timestamp`, `RecordingUrl` (if applicable).

## Idempotency

The REST API is *not* automatically idempotent for `POST /Messages`. Two retries will send two SMS. Recommended pattern:

- Generate a `client_message_id` (UUID) per logical send.
- Store `(client_message_id, MessageSid)` after first successful create; short-circuit on retry.
- For Programmable Voice, the `Calls` resource accepts `RecordingChannels`/`Trim` etc. but no native idempotency key — same dedup-on-your-side rule applies.

## Webhook handler shape

```python
@app.post("/twilio/sms-status", dependencies=[Depends(verify_twilio)])
async def sms_status(form: dict = Depends(form_to_dict)):
    sid    = form["MessageSid"]
    status = form["MessageStatus"]
    code   = form.get("ErrorCode")
    await store.update_status(sid, status, code)
    return Response(status_code=204)     # Twilio ignores body but expects 2xx
```

⚠️ Always return 2xx fast (≤ ~10s). Twilio retries 5xx with backoff; persistent failures fill your debugger and may auto-disable the webhook.
