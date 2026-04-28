# VOX 01 — Outbound Calls

🔑 `client.calls.create(to=…, from_=…, url=…)` dials a number; Twilio fetches **TwiML** from `url` to drive what happens once it answers.

## Minimum

```python
call = client.calls.create(
    to="+15551234567",
    from_="+15557654321",                       # must be a Twilio number or verified caller ID
    url="https://app.example.com/twiml/answer",
)
print(call.sid, call.status)  # CA..., queued
```

Twilio dials. When the recipient answers, Twilio HTTP-POSTs to `url`. The TwiML you return drives the call: speak, gather, dial another party, record, hang up.

## Inline TwiML (skip the webhook)

For one-shot calls where the script is fully known up front:

```python
twiml = """
<Response>
  <Say voice="Polly.Joanna">Hi — your appointment is at 3pm.</Say>
  <Hangup/>
</Response>
""".strip()

client.calls.create(
    to="+15551234567",
    from_="+15557654321",
    twiml=twiml,
)
```

`twiml` is mutually exclusive with `url`. Inline is fine for static notifications; use `url` for anything dynamic.

## Useful create parameters

| Param | Purpose |
|---|---|
| `Url` / `Method` | Where Twilio fetches TwiML on answer. |
| `FallbackUrl` / `FallbackMethod` | If the primary URL 5xx's. |
| `StatusCallback` + `StatusCallbackEvent` | Lifecycle webhooks: `initiated`, `ringing`, `answered`, `completed`. |
| `Timeout` | Ring duration (default 60s). |
| `Record` | `true` → record from answer to hangup. |
| `RecordingChannels` | `mono` or `dual`. Dual = caller / callee on separate channels (essential for transcription quality). |
| `RecordingStatusCallback` | URL fired once recording is processed. |
| `MachineDetection` | `Enable` or `DetectMessageEnd`. Tells you in the answer webhook whether a human or voicemail picked up (`AnsweredBy=human|machine_*`). |
| `MachineDetectionTimeout` | Cap on detection window. |
| `AsyncAmd` | Run AMD asynchronously and report via `AsyncAmdStatusCallback`. |
| `Trim` | `trim-silence` to remove dead air from recordings. |
| `SipAuthUsername` / `SipAuthPassword` | When `to` is a SIP URI requiring auth. |
| `CallerId` | Override `From` for caller-ID display (must be verified). |
| `ApplicationSid` | Use a TwiML App's URL instead of inline. |

## Status callbacks

```python
client.calls.create(
    to="+15551234567",
    from_="+15557654321",
    url="https://app.example.com/twiml/answer",
    status_callback="https://app.example.com/twilio/call-status",
    status_callback_event=["initiated", "ringing", "answered", "completed"],
    status_callback_method="POST",
)
```

Webhook body fields: `CallSid`, `CallStatus` (`queued`/`ringing`/`in-progress`/`completed`/`busy`/`no-answer`/`failed`/`canceled`), `CallDuration`, `Direction`, `From`, `To`, `Timestamp`. Recording-related events arrive on `RecordingStatusCallback`, not the call-status URL.

## Modify a live call

```python
client.calls("CA...").update(
    twiml="<Response><Say>Transferring you now.</Say><Dial>+15558675309</Dial></Response>",
)
```

Pushing new TwiML interrupts whatever the call is currently doing. Useful for "warm transfer" patterns from a supervisor UI.

Hang up by force:

```python
client.calls("CA...").update(status="completed")
```

## Caller ID rules

`From` must be:

- A Twilio phone number on the account, or
- A **Verified Caller ID** (phone you proved ownership of via inbound call), or
- A **SIP** address you own.

For US originating traffic, **SHAKEN/STIR** attestation is automatic for Twilio numbers; spoofed `From` (caller-ID rewriting to a non-owned number) is blocked.

## Answering Machine Detection (AMD)

```python
client.calls.create(
    to="+15551234567",
    from_="+15557654321",
    url="https://app.example.com/twiml/answer",
    machine_detection="DetectMessageEnd",
    machine_detection_timeout=10,
)
```

Twilio detects voicemail and waits for the beep before invoking your TwiML, so your `<Say>` lands *as the voicemail message*. The answer webhook gets `AnsweredBy` ∈ `human`, `machine_start`, `machine_end_beep`, `machine_end_silence`, `machine_end_other`, `fax`, `unknown`.

⚠️ AMD adds 1–4s of latency before TwiML executes.

## Pricing-aware patterns

- Fail fast: short `Timeout` for outbound dialers.
- Don't `<Pause>` in queued calls — you pay per second on bridge.
- Use `AsyncAmd=true` + your own logic to avoid the 1–4s amd block when the caller experience matters.
