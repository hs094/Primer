# VOX 08 — Media Streams

🔑 **Media Streams** = a WebSocket connected to a live call that carries the audio bytes both ways. The lever for plugging real-time STT, LLM, and custom DSP into a Twilio voice call.

## Use cases

- Real-time speech-to-text (Whisper-stream, Deepgram, AssemblyAI live).
- Conversational AI agents (Twilio's own ConversationRelay, or roll your own with OpenAI Realtime / Gemini Live).
- Custom hold music, audio injection, voice changers.
- Compliance recording into your own storage.

## TwiML

Two flavours — pick based on whether you want the call to *also* run TwiML:

### `<Start><Stream>` — mirror audio while TwiML continues

```xml
<Response>
  <Start>
    <Stream url="wss://app.example.com/audio" track="inbound_track"/>
  </Start>
  <Say>Connecting you to support.</Say>
  <Dial>+15558675309</Dial>
</Response>
```

Audio is mirrored to your WebSocket; the call still does whatever the rest of the TwiML says. Good for "transcribe in the background while the IVR runs".

### `<Connect><Stream>` — stream IS the call

```xml
<Response>
  <Connect>
    <Stream url="wss://app.example.com/audio">
      <Parameter name="caller_id" value="alice"/>
    </Stream>
  </Connect>
</Response>
```

`<Connect>` blocks the TwiML — the WebSocket *is* the call experience. You drive everything (TTS, STT, agent logic) over the socket. Twilio plays whatever audio you send back.

`track`: `inbound_track`, `outbound_track`, or `both_tracks`.

## Audio format

- **µ-law** (PCM-mu-law), 8 kHz, mono. Each frame ≈ 20 ms (160 bytes payload).
- Bytes are **base64-encoded** inside JSON messages over the WebSocket.

## WebSocket protocol

Twilio sends/expects JSON messages.

### Inbound (Twilio → you)

```json
{ "event": "connected", "protocol": "Call", "version": "1.0.0" }

{ "event": "start",
  "sequenceNumber": "1",
  "start": {
    "streamSid": "MZ…",
    "accountSid": "AC…",
    "callSid":   "CA…",
    "tracks":    ["inbound"],
    "mediaFormat": { "encoding": "audio/x-mulaw", "sampleRate": 8000, "channels": 1 },
    "customParameters": { "caller_id": "alice" }
  } }

{ "event": "media",
  "sequenceNumber": "42",
  "media": { "track": "inbound", "chunk": "42", "timestamp": "8400", "payload": "<base64 µ-law 160 bytes>" } }

{ "event": "stop", "sequenceNumber": "999", "stop": { "callSid": "CA…", "accountSid": "AC…" } }
```

### Outbound (you → Twilio)

Send audio back into the call:

```json
{ "event": "media",
  "streamSid": "MZ…",
  "media": { "payload": "<base64 µ-law>" } }
```

Send a synthetic mark to know when Twilio finished playing your audio:

```json
{ "event": "mark", "streamSid": "MZ…", "mark": { "name": "tts-done" } }
```

Clear queued audio (for barge-in / interruption):

```json
{ "event": "clear", "streamSid": "MZ…" }
```

## Python skeleton (FastAPI)

```python
import base64, json
from fastapi import FastAPI, WebSocket

app = FastAPI()

@app.websocket("/audio")
async def audio(ws: WebSocket) -> None:
    await ws.accept()
    stream_sid: str | None = None
    async for raw in ws.iter_text():
        msg = json.loads(raw)
        match msg["event"]:
            case "start":
                stream_sid = msg["start"]["streamSid"]
            case "media":
                pcm_mulaw = base64.b64decode(msg["media"]["payload"])
                await stt.feed(pcm_mulaw)              # your STT
            case "stop":
                break

        # to speak: synthesise µ-law and send back
        if (chunk := await tts.next_chunk()) is not None:
            await ws.send_text(json.dumps({
                "event": "media",
                "streamSid": stream_sid,
                "media": {"payload": base64.b64encode(chunk).decode()},
            }))
```

## Latency & throughput

- 20 ms frames, 50 fps. End-to-end round-trip target: **< 800 ms** for natural conversational AI.
- Twilio buffers your outbound audio before playing — clear with `event: clear` to support barge-in.
- Backpressure: Twilio drops frames if your WS reader stalls. Read aggressively, do work elsewhere.

## ConversationRelay (managed alternative)

If you don't want to wire your own STT/TTS/LLM glue, Twilio's **ConversationRelay** (v1) provides a managed pipeline:

```xml
<Connect>
  <ConversationRelay url="wss://app.example.com/relay"
                      welcomeGreeting="Hi, how can I help?"
                      voice="Polly.Joanna"
                      transcriptionProvider="Deepgram"/>
</Connect>
```

Twilio runs STT + TTS; your WebSocket only sees text events (`prompt`, `interrupt`) and emits text deltas. Much less plumbing for typical voice-AI agents.

## Pricing

- Streaming itself adds a small per-minute fee on top of the call leg.
- The expensive part is your downstream STT / LLM provider — budget separately.
