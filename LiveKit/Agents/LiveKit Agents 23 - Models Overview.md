# 23 — Models Overview

🔑 LiveKit Inference is a managed multi-provider gateway; plugins are open-source per-provider integrations — both plug into the same `AgentSession` slots.

Source: https://docs.livekit.io/agents/models/

## Two Access Methods
- **LiveKit Inference** — Built-in access to OpenAI, Google, AssemblyAI, Deepgram, Cartesia, ElevenLabs, etc. No separate API keys; billing via LiveKit Cloud.
- **Plugins** — Install `livekit-agents[<provider>]` extras for direct provider access with your own credentials.

```bash
uv add "livekit-agents[openai,deepgram,cartesia,silero]~=1.4"
```

## Four Model Categories
| Slot | Role |
| --- | --- |
| `stt` | Speech-to-text transcription |
| `llm` | Reasoning + tool calls |
| `tts` | Text-to-speech synthesis |
| `RealtimeModel` | Direct speech-to-speech (replaces stt+llm+tts) |

## Combined Example
```python
from livekit.agents import AgentSession, inference

session = AgentSession(
    stt=inference.STT(model="deepgram/flux-general"),
    llm=inference.LLM(model="openai/gpt-5.3-chat-latest"),
    tts=inference.TTS(model="cartesia/sonic-3", voice="..."),
)
```

You can mix Inference and plugins freely, and swap models mid-session (e.g., escalate to a stronger LLM after intent detection).

## Model Identifiers
Inference uses `provider/model[:variant]` strings, e.g. `"deepgram/nova-3:multi"`, `"openai/gpt-5.3-chat-latest"`, `"cartesia/sonic-3"`. The special `"auto:<lang>"` lets LiveKit pick.

## Tags
[[LiveKit]] [[Agents]] [[Models]] [[Inference]] [[STT]] [[LLM]] [[TTS]]
