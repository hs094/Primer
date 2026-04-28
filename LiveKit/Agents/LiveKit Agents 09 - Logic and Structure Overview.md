# 09 — Logic and Structure Overview

🔑 An agent server registers with LiveKit, dispatches a per-job subprocess that joins a room, and runs an `AgentSession` orchestrating STT-LLM-TTS over WebRTC.

Source: https://docs.livekit.io/agents/build/

## Runtime model
- **Agent server**: long-running Python/Node process registered with LiveKit. Waits for dispatch.
- **Job subprocess**: spawned per dispatch; isolates each session. Crash in one job does not affect others.
- **Entrypoint**: main function for each job (decorated in Node; in Python, the function passed to `WorkerOptions(entrypoint_fnc=...)`). Receives a `JobContext`.
- **Transport**: WebRTC between frontend and agent; HTTP/WebSocket from agent to backend services.

## Building blocks
- **`AgentSession`** — orchestrator for a single conversation. Wires STT, LLM, TTS, VAD, turn detector.
- **`Agent`** — long-lived role; holds `instructions`, tools, and lifecycle hooks. One agent is "active" per session; handoffs swap the active agent.
- **Tasks** — short-lived, return a typed result; used for discrete sub-flows.
- **Pipeline nodes** — overridable stages: `stt_node`, `llm_node`, `tts_node`, `transcription_node`, plus realtime audio nodes.
- **Hooks** — `on_enter`, `on_exit`, `on_user_turn_completed`.

## Capability surface
Streaming STT-LLM-TTS, turn detection, interruption handling, function tools, multi-agent handoffs, voice + video + text modalities. Apache 2.0; first-party plugins for OpenAI, Google, Azure, AWS, Deepgram, etc.

## Tags
[[LiveKit]] [[Agents]] [[AgentSession]] [[Workers]] [[Architecture]]
