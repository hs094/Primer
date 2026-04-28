# 25 — STT Integrations

🔑 STT turns user audio into text for the LLM; pick a streaming model and pair it with VAD for fastest turn detection.

Source: https://docs.livekit.io/agents/integrations/stt/

## LiveKit Inference (managed)
- AssemblyAI — Universal-3 Pro Streaming, Universal-Streaming
- Cartesia — Ink Whisper (100 langs)
- Deepgram — Flux, Nova-2, Nova-3
- ElevenLabs — Scribe v2 Realtime (190 langs)
- xAI — Speech to Text (25 langs)

```python
from livekit.agents import AgentSession, inference

session = AgentSession(
    stt=inference.STT(model="deepgram/nova-3:multi"),  # multilingual
)
# or let LiveKit choose
session = AgentSession(stt=inference.STT(model="auto:es"))
```

## Plugins (BYO key)
20+ providers including AWS Transcribe, Azure AI Speech, Google Cloud Speech, OpenAI Whisper, Deepgram, AssemblyAI, Speechmatics, Gladia, fal, Sarvam.

```python
from livekit.plugins import deepgram, openai, assemblyai

stt = deepgram.STT(model="nova-3", language="multi")
stt = openai.STT(model="whisper-1")
stt = assemblyai.STT()
```

## Adapters
- `StreamAdapter` — wrap a non-streaming STT with VAD to fake streaming.
- `MultiSpeakerAdapter` — diarization for multi-speaker rooms.

```python
from livekit.agents.stt import StreamAdapter
from livekit.plugins import openai, silero

stt = StreamAdapter(stt=openai.STT(), vad=silero.VAD.load())
```

## Language Codes
ISO 639-1 (`"en"`), BCP-47 (`"en-US"`), names (`"english"`), and underscored (`"en_us"`) all normalize.

## Tags
[[LiveKit]] [[Agents]] [[STT]] [[Deepgram]] [[VAD]] [[Inference]]
