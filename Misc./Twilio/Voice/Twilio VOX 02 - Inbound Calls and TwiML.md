# VOX 02 — Inbound Calls and TwiML

🔑 Inbound call → Twilio POSTs to your number's `voice_url` → you return **TwiML** describing what to do. Each verb plays sequentially; flow can branch via `<Gather>` + a follow-up webhook.

## Webhook payload (inbound)

| Field | Notes |
|---|---|
| `CallSid` | `CA…` |
| `From` | Caller's number (E.164). |
| `To` | Your Twilio number. |
| `CallStatus` | Usually `ringing` on first hit. |
| `Direction` | `inbound`. |
| `CallerName` | If you bought CNAM / Lookup. |
| `FromCity`, `FromState`, `FromCountry`, `FromZip` | Best-effort geo. |
| `ApiVersion` | `2010-04-01`. |
| `AccountSid` | `AC…` |

## Hello-world TwiML

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Response>
  <Say voice="Polly.Joanna">Hi! Thanks for calling.</Say>
  <Hangup/>
</Response>
```

## Build with the Python SDK

```python
from fastapi import APIRouter
from fastapi.responses import Response
from twilio.twiml.voice_response import VoiceResponse, Gather

router = APIRouter()

@router.post("/twilio/voice")
async def voice_inbound() -> Response:
    twiml = VoiceResponse()
    gather = Gather(
        input="dtmf speech",
        action="/twilio/voice/menu",
        method="POST",
        num_digits=1,
        timeout=5,
        speech_timeout="auto",
    )
    gather.say("Press 1 for sales, 2 for support.")
    twiml.append(gather)
    twiml.say("We didn't get that — goodbye.")
    twiml.hangup()
    return Response(str(twiml), media_type="application/xml")


@router.post("/twilio/voice/menu")
async def voice_menu(Digits: str = Form(""), SpeechResult: str = Form("")) -> Response:
    twiml = VoiceResponse()
    if Digits == "1" or "sales" in SpeechResult.lower():
        twiml.dial("+15558675309")
    elif Digits == "2" or "support" in SpeechResult.lower():
        twiml.dial("+15558675310")
    else:
        twiml.redirect("/twilio/voice")
    return Response(str(twiml), media_type="application/xml")
```

⚠️ Don't return JSON. Set `Content-Type: application/xml` (or `text/xml`).

## Common verb chains

### Voicemail

```python
twiml = VoiceResponse()
twiml.say("Leave a message after the beep.")
twiml.record(
    max_length=120,
    finish_on_key="#",
    transcribe=True,
    transcribe_callback="/twilio/transcript",
    recording_status_callback="/twilio/recording",
)
twiml.hangup()
```

### Forward to mobile, then voicemail

```python
twiml = VoiceResponse()
dial = twiml.dial(action="/twilio/voice/after-dial", timeout=20)
dial.number("+15558675309")
# action fires whether dial was answered or not — DialCallStatus tells us
```

In `/twilio/voice/after-dial`, check `DialCallStatus` (`completed`, `busy`, `no-answer`, `failed`) and either `<Hangup/>` or `<Record>` voicemail.

### Bridge to another agent (warm)

```python
twiml = VoiceResponse()
twiml.say("Connecting you.")
twiml.dial(answer_on_bridge=True).client("agent_alice")
```

`answerOnBridge=true` keeps the inbound leg in `ringing` until the second leg answers — caller hears ringback, not silence.

### IVR with speech

```python
gather = Gather(input="speech", action="/twilio/voice/route", speech_timeout="auto", hints="appointment, billing, hours")
gather.say("How can I help?")
twiml.append(gather)
```

`hints` biases the recogniser toward your domain words.

## Fallback URL

If your `voice_url` 5xx's or times out (>15s without TwiML), Twilio hits `voice_fallback_url`. Always set one — even a static "we're having issues, please call back" TwiML beats a dropped call.

## Returning empty / silent

```xml
<Response/>
```

…hangs up immediately. Useful if you want to reject without a `<Reject>` (which sounds like a busy tone).

## Webhook timing

- Twilio aborts after **15 seconds** without response → fallback URL.
- Long downstream calls (DB, LLM) belong outside the webhook: `<Pause>` won't help; use `<Redirect>` to a "thinking" loop or speak a placeholder while you compute.
