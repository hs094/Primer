# VOX 05 — Recording

🔑 You can record from REST (`Record=true` on `calls.create`), inline TwiML (`<Record>`), or `<Dial record="…">`. Each produces a `Recording` resource (`RE…`) once the call ends.

## Three ways to start a recording

### 1. REST — record entire call

```python
client.calls.create(
    to="+15551234567",
    from_="+15557654321",
    url="https://app.example.com/twiml",
    record=True,
    recording_channels="dual",                    # caller + callee on separate channels
    recording_status_callback="/twilio/recording",
    recording_status_callback_event=["completed", "absent", "failed"],
    trim="trim-silence",
)
```

Recording starts when the call connects, ends on hangup.

### 2. TwiML `<Record>` verb — capture caller

```xml
<Response>
  <Say>Leave a message.</Say>
  <Record maxLength="120" finishOnKey="#" playBeep="true" transcribe="true"
          transcribeCallback="/twilio/transcript"
          recordingStatusCallback="/twilio/recording"/>
</Response>
```

Records only the caller (one-sided). Useful for voicemail.

### 3. TwiML `<Dial record="…">` — record both legs

```xml
<Response>
  <Dial record="record-from-answer-dual"
        recordingStatusCallback="/twilio/recording">
    <Number>+15558675309</Number>
  </Dial>
</Response>
```

Modes:

| Mode | Includes |
|---|---|
| `do-not-record` | nothing |
| `record-from-ringing` | from dialing through hangup |
| `record-from-answer` | from answer through hangup |
| `record-from-ringing-dual` | dual-channel from dialing |
| `record-from-answer-dual` | dual-channel from answer |

⚠️ **Always prefer dual** for transcription / agent-vs-customer analytics. Mono mixes both speakers, breaking diarisation.

## Recording resource

`RE…`. Endpoint:

```
https://api.twilio.com/2010-04-01/Accounts/{AccountSid}/Recordings/{RecordingSid}.json
```

Fields: `sid`, `call_sid`, `conference_sid`, `duration`, `channels` (1 or 2), `status` (`processing` → `completed` / `absent` / `failed`), `source` (`DialVerb`, `RecordVerb`, `OutboundAPI`, `Conference`, `Trunking`, `RecordingController`), `price`, `price_unit`, `media_url`, `start_time`, `encryption_details`.

```python
rec = client.recordings("RE...").fetch()
print(rec.duration, rec.channels)
```

## Download

```python
import httpx, os
auth = (os.environ["TWILIO_ACCOUNT_SID"], os.environ["TWILIO_AUTH_TOKEN"])
url = f"https://api.twilio.com{rec.uri.replace('.json', '.mp3')}"

async with httpx.AsyncClient(auth=auth, follow_redirects=True) as c:
    audio = (await c.get(url)).content
```

Format choices: `.mp3` (compressed), `.wav` (PCM 8 kHz). Append `.wav` for STT pipelines that prefer raw audio.

## Status callback

Posted when recording is ready:

| Field | Value |
|---|---|
| `RecordingSid` | `RE…` |
| `RecordingUrl` | URL to fetch (no extension; add `.mp3`/`.wav`). |
| `RecordingStatus` | `completed`, `failed`, `absent`. |
| `RecordingDuration` | Seconds. |
| `RecordingChannels` | `1` or `2`. |
| `RecordingSource` | as above. |
| `CallSid` | parent call. |

## Transcription

Two paths:

1. **TwiML `<Record transcribe=true>`** — Twilio's built-in transcription. Cheap-and-cheerful, English-only, single-channel quality. Result posted to `transcribeCallback` with `TranscriptionText`.
2. **Voice Intelligence / Conversational Intelligence** — newer, language-aware, sentiment + entities + summarisation. Submit a recording to a Service:

```python
client.intelligence.v2.transcripts.create(
    service_sid="GA...",
    channel={"media_properties": {"source_sid": "RE..."}},
)
```

3. **External STT** (Whisper, Deepgram, AssemblyAI) — fetch the recording, post to your STT, store the result. Most production teams do this.

## Encryption at rest

For PCI / HIPAA: enable **Voice Recording Encryption**. You upload an X.509 public key per account; Twilio encrypts each recording with a per-recording symmetric key, then encrypts that key with your public key (`encryption_details` on the resource). You decrypt locally. Recordings stay opaque to Twilio Support.

## Pause / resume / split

Mid-call, the **Recording Controller** REST API lets you pause, resume, or split:

```python
client.calls("CA...").recordings.create(recording_status_callback="/cb")     # start mid-call
client.calls("CA...").recordings("RE...").update(status="paused")            # pause / resume / stopped
```

Pause for sensitive moments (PCI card capture is one — but use `<Pay>` for that).

## Retention & deletion

Default retention is per Console settings. Delete proactively:

```python
client.recordings("RE...").delete()
```

⚠️ Audio is deleted; the metadata (`RE…` + duration + price) persists for billing.

## Compliance reminders

- Two-party consent states (CA, FL, IL, …) — say "this call is being recorded" before recording starts.
- HIPAA — sign Twilio's BAA *and* enable encryption.
- GDPR — recordings are personal data; document retention and DSAR processes.
