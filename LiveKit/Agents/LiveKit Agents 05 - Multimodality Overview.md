# 05 — Multimodality Overview

🔑 A single `AgentSession` can ingest and emit speech, text, and vision concurrently — agents see, read, hear, and speak in the same turn.

Source: https://docs.livekit.io/agents/multimodality/ (canonical; `/agents/build/multimodal/` 404s)

## Supported modalities
- **Speech & audio** — realtime STT, VAD-driven turn detection, interruption handling, TTS output
- **Text & transcriptions** — text-only sessions, hybrid voice+text, live transcript streams
- **Images & video** — static images in chat context, sampled video frames, live video to a realtime model, virtual avatar output

## Mental model
> "Agents can process and generate speech, text, images, and live video, allowing them to understand context from different sources and respond in the most appropriate format."

Each input/output has an enable/disable toggle on the session, so you can drive a single agent across channels.

## Toggling modalities at runtime
```python
# disable audio for a text-only stretch
session.input.set_audio_enabled(False)
session.output.set_audio_enabled(False)

# re-enable when the user wants to talk again
session.input.set_audio_enabled(True)
session.output.set_audio_enabled(True)
```

## Typical pairings
- Voice assistant: speech in, speech + transcription out
- Visual assistant: live video + speech in, speech out
- Chatbot: text in, text out (`AgentSession` with no STT/TTS)
- Avatar agent: speech in, speech + avatar video out

## Tags
[[LiveKit]] [[Agents]] [[Multimodal]] [[Vision]] [[Speech]]
