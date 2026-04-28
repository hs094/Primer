# 27 — Realtime Models

🔑 Realtime models consume and emit speech directly, collapsing STT+LLM+TTS into one slot — lower latency, less control over prosody.

Source: https://docs.livekit.io/agents/integrations/realtime/

## Providers
- Amazon Nova Sonic (Python only)
- Azure OpenAI Realtime API (Python + Node)
- Google Gemini Live API (Python + Node)
- NVIDIA PersonaPlex (Python only)
- OpenAI Realtime API (Python + Node)
- Phonic Speech-to-speech (Python + Node)
- Ultravox Realtime (Python only)
- xAI Grok Voice Agent API (Python + Node)

## Basic Usage
```python
from livekit.agents import AgentSession
from livekit.plugins import openai

session = AgentSession(llm=openai.realtime.RealtimeModel())
```

```python
from livekit.plugins import google

session = AgentSession(
    llm=google.beta.realtime.RealtimeModel(model="gemini-2.5-flash-preview-native-audio-dialog"),
)
```

## Limitations
- No interim transcription events.
- Cannot synthesize from arbitrary text scripts mid-turn.
- Emotional/contextual cues are weak when chat history is loaded as plain text.

## Half-Cascade Pattern
Use a realtime model for understanding + interruption handling, but route output through a dedicated TTS for voice control:

```python
session = AgentSession(
    llm=openai.realtime.RealtimeModel(modalities=["text"]),
    tts=inference.TTS(model="cartesia/sonic-3", voice="..."),
)
```

This gives you precise voice/style at the cost of a small latency hit.

## Tags
[[LiveKit]] [[Agents]] [[RealtimeModel]] [[OpenAI]] [[Gemini]] [[SpeechToSpeech]]
