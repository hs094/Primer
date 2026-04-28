# VOX 03 — TwiML Voice Reference

🔑 TwiML is **XML-as-DSL** for voice. Verbs are commands; nouns nest inside verbs to qualify them. Twilio executes verbs in document order.

## Verb cheatsheet

### `<Say>` — text-to-speech

```xml
<Say voice="Polly.Joanna" language="en-US" loop="2">Welcome.</Say>
```

| Attr | Default | Notes |
|---|---|---|
| `voice` | `Polly.Joanna` (en-US) | Amazon Polly voices recommended (`Polly.<Name>`); `man`/`woman`/`alice` legacy. |
| `language` | inferred from voice | e.g. `en-US`, `en-GB`, `es-ES`, `de-DE`. |
| `loop` | `1` | Repeat N times. `0` = repeat forever (don't). |

Supports SSML inside `<Say>`: `<break time="500ms"/>`, `<emphasis>`, `<say-as interpret-as="digits">`, `<phoneme>`, `<prosody>`.

### `<Play>` — play audio file

```xml
<Play loop="1" digits="1234">https://app.example.com/hold.mp3</Play>
```

- File must be MP3 or WAV (8 kHz / 16 kHz mono recommended).
- `digits` plays DTMF tones instead of audio (subdialing pattern).

### `<Pause>`

```xml
<Pause length="2"/>   <!-- seconds, default 1 -->
```

### `<Gather>` — capture DTMF / speech

```xml
<Gather input="dtmf speech"
        action="/twiml/menu"
        method="POST"
        timeout="5"
        numDigits="1"
        finishOnKey="#"
        speechTimeout="auto"
        hints="billing, sales, support"
        partialResultCallback="/twiml/partial"
        language="en-US"
        speechModel="experimental_conversations"
        actionOnEmptyResult="false">
  <Say>How can I help?</Say>
</Gather>
```

| Attr | Notes |
|---|---|
| `input` | `dtmf`, `speech`, or `dtmf speech`. |
| `action` | URL receives `Digits` or `SpeechResult` + `Confidence`. |
| `numDigits` | Auto-submit when reached. |
| `finishOnKey` | Default `#`. |
| `timeout` | Inter-digit silence timeout (s). |
| `speechTimeout` | `auto` = end-of-utterance detection. |
| `hints` | Comma-separated bias terms. |
| `speechModel` | `default`, `numbers_and_commands`, `phone_call`, `experimental_conversations`. |
| `partialResultCallback` | Fires with interim transcripts (latency win). |

If the user provides nothing, control falls through past `<Gather>` to the next verb — useful for re-prompting.

### `<Dial>` — connect another party

```xml
<Dial action="/twiml/after"
      timeout="30"
      callerId="+15557654321"
      record="record-from-answer-dual"
      answerOnBridge="true"
      hangupOnStar="true">
  <Number>+15558675309</Number>
</Dial>
```

| Attr | Notes |
|---|---|
| `timeout` | Ring duration (s). |
| `callerId` | Override displayed caller (must be verified). |
| `record` | `do-not-record` \| `record-from-answer` \| `record-from-ringing` \| `*-dual`. |
| `answerOnBridge` | `true` = keep A-leg ringing until B-leg answers; better caller UX. |
| `hangupOnStar` | A-leg presses `*` to drop B-leg. |
| `action` | Posted with `DialCallStatus` once the dialled leg ends. |
| `recordingStatusCallback` | URL for the recording resource. |
| `referUrl` / `sequential` | SIP REFER and ordered fan-out. |

Dial **nouns** (mutually exclusive children of `<Dial>`):

| Noun | Dials |
|---|---|
| `<Number>+15551234567</Number>` | PSTN number. Attrs: `url` (whisper TwiML to callee), `sendDigits`. |
| `<Sip>sip:user@host:5060;transport=tls</Sip>` | SIP URI. Attrs: `username`, `password`. |
| `<Client>agent_alice</Client>` | Twilio Client SDK identity. |
| `<Conference>RoomName</Conference>` | Multi-party conference. See [[Voice/Twilio VOX 06 - Conferences]]. |
| `<Queue>support</Queue>` | Pull from a queue (Bridge to oldest waiter). |

Multiple `<Number>` children = parallel ring; first to answer wins.

### `<Record>`

```xml
<Record maxLength="120"
        finishOnKey="#"
        timeout="5"
        playBeep="true"
        trim="trim-silence"
        recordingChannels="dual"
        recordingStatusCallback="/twiml/recording"
        transcribe="true"
        transcribeCallback="/twiml/transcript"/>
```

After recording stops, Twilio posts `RecordingSid`, `RecordingUrl`, `RecordingDuration` to the status callback. The audio lives at `https://api.twilio.com/.../Recordings/RE…[.mp3|.wav]` (basic-auth-protected).

### `<Hangup/>`

End call immediately.

### `<Redirect>`

```xml
<Redirect method="POST">/twiml/menu</Redirect>
```

Twilio fetches new TwiML at the URL — flow continues there. Useful for shared subroutines.

### `<Reject>`

```xml
<Reject reason="busy"/>   <!-- or reason="rejected" -->
```

Decline an inbound call without billing for an answer.

### `<Enqueue>`

```xml
<Enqueue waitUrl="/twiml/hold-music" action="/twiml/post-queue">support</Enqueue>
```

Adds caller to named queue; agents pull via `<Dial><Queue>support</Queue></Dial>`. `waitUrl` plays while waiting (TwiML).

### `<Refer>`

SIP REFER for transferring a SIP call to a third party without media bridging through Twilio.

### `<Stream>`

```xml
<Connect>
  <Stream url="wss://app.example.com/audio"/>
</Connect>
```

Bidirectional **Media Stream** — raw audio frames over WebSocket. See [[Voice/Twilio VOX 08 - Media Streams]]. Use for real-time STT/TTS / agent integrations (LLMs).

`<Connect>` (full takeover) vs `<Start>` (mirror to stream while the call continues normally).

### `<Pay>`

```xml
<Pay paymentConnector="Stripe_Connector_1"
     chargeAmount="42.00"
     currency="USD"
     description="Booking #1234"
     action="/twiml/pay-result"/>
```

PCI-compliant card capture during the call (DTMF). Twilio handles tokenisation; you receive a token for your processor.

### `<Say-as>` (SSML inside `<Say>`)

```xml
<Say><say-as interpret-as="digits">12345</say-as></Say>
```

`interpret-as`: `digits`, `cardinal`, `ordinal`, `date`, `time`, `telephone`, `address`, `expletive`, `verbatim`.

## Document model

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Response>
  <!-- verbs go here, in order -->
</Response>
```

Verbs run sequentially. `<Gather>` and `<Dial>` are *blocking* (wait for input or for the dialled leg to end). Returning a fresh TwiML doc on `action` URLs is the standard branching mechanism.
