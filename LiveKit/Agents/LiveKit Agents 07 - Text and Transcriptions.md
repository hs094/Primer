# 07 — Text and Transcriptions

🔑 Transcriptions stream on the `lk.transcription` topic synced to TTS playout; the `lk.chat` topic carries text-mode I/O — the same `AgentSession` handles both.

Source: https://docs.livekit.io/agents/build/text/

## Transcription stream
Live captions publish to the `lk.transcription` text stream topic. By default, words appear in sync with TTS audio.

```python
from livekit.agents import AgentSession, room_io

session = AgentSession(
    ...,
    room_output_options=room_io.RoomOutputOptions(
        sync_transcription=False,   # stream as soon as text is generated
    ),
)
```

## TTS-aligned word timings
Cartesia and ElevenLabs return per-word timestamps. Opt in for tighter sync:
```python
session = AgentSession(
    ...,
    use_tts_aligned_transcript=True,
)
```

## Text transforms before TTS
```python
session = AgentSession(
    ...,
    tts_text_transforms=["filter_markdown", "filter_emoji"],
)
```
Custom transform — a callable over a streaming text iterator:
```python
async def upper(stream):
    async for chunk in stream:
        yield chunk.upper()

session = AgentSession(..., tts_text_transforms=[upper])
```

## Text input
- Frontends send via the SDK's `sendText()` on the `lk.chat` topic
- Server-side injection: `session.generate_reply(user_input="...")`
- Custom callbacks let you intercept chat messages for command parsing or filtering

## Text-only session
```python
session = AgentSession(llm=openai.LLM(model="gpt-5.3-chat"))  # no STT/TTS
```

## Toggle audio at runtime
```python
session.input.set_audio_enabled(False)
session.output.set_audio_enabled(False)
```

## Tags
[[LiveKit]] [[Agents]] [[Text]] [[Transcriptions]] [[Cartesia]] [[ElevenLabs]]
