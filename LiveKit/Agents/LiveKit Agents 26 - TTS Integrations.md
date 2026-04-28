# 26 — TTS Integrations

🔑 TTS streams synthesized audio back to the room; choose a low-latency streaming voice (Cartesia Sonic, Deepgram Aura, ElevenLabs Flash) for responsive agents.

Source: https://docs.livekit.io/agents/integrations/tts/

## LiveKit Inference (managed)
- Cartesia — Sonic, Sonic-2, Sonic-3
- Deepgram — Aura, Aura-2
- ElevenLabs — Flash, Turbo, v3 variants
- Inworld
- Rime — Arcana, Mist
- xAI

```python
from livekit.agents import AgentSession, inference

session = AgentSession(
    tts=inference.TTS(
        model="cartesia/sonic-3",
        voice="9626c31c-bec5-4cca-baa8-f8ba9e84c8bc",
        language="en",
    ),
)
```

## Plugins (BYO key)
30+ providers: Amazon Polly, Azure AI Speech, Google Cloud TTS, OpenAI, ElevenLabs, Cartesia, PlayHT, Hume, Resemble, Neuphonic, Speechify, Lmnt, etc.

```python
from livekit.plugins import cartesia, elevenlabs, openai

tts = cartesia.TTS(model="sonic-3", voice="...")
tts = elevenlabs.TTS(voice_id="...", model="eleven_flash_v2_5")
tts = openai.TTS(voice="alloy")
```

## Standalone Streaming
Push text, consume audio frames — no `AgentSession` required.

```python
stream = tts.stream()
stream.push_text("Hello there.")
stream.flush()
async for ev in stream:
    audio_frame = ev.frame  # rtc.AudioFrame
```

## Language Normalization
Same as STT: `"en"`, `"en-US"`, `"eng"`, `"english"`, `"en_us"` all resolve.

## Tags
[[LiveKit]] [[Agents]] [[TTS]] [[Cartesia]] [[ElevenLabs]] [[Inference]]
