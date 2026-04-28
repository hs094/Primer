# 01 — WebRTC Overview

🔑 LiveKit is an SFU-based WebRTC platform with a Room/Participant/Track abstraction tuned for voice, video, and AI-agent workloads.

Source: https://docs.livekit.io/home/get-started/intro-to-livekit/

## What LiveKit Is
Open-source framework + cloud platform for real-time voice, video, and physical-AI agents. Wraps WebRTC with a managed SFU (Selective Forwarding Unit) so any participant — browser, mobile, server, or AI agent — can publish and subscribe over a single room session.

## Core Model
- `Room` — logical session, identified by name in the JWT.
- `Participant` — `LocalParticipant` (this client) and `RemoteParticipant` (others).
- `Track` — audio, video, or data stream owned by a participant.
- `TrackPublication` — server-side metadata; subscribe to materialize into a `Track`.

## Transport Architecture
WebRTC media (RTP/SRTP) flows client ↔ SFU; the SFU forwards selected layers per subscriber. Signaling rides a WebSocket (`wss://...`) used for room state, RPC, data packets, and ICE/SDP exchange. JWT access tokens authenticate the WebSocket handshake.

## SDK Surface
Client SDKs: JavaScript/TypeScript, Swift, Android, Flutter, React Native, Unity.
Server SDKs: Go, Node.js, Python, Ruby, Rust.
Agents framework (Python, Node.js) lets backend processes act as full participants.

## Typical Pipelines
- Voice agent: user mic → SFU → agent STT → LLM → TTS → SFU → user speaker.
- Conference: many publishers, many subscribers, simulcast layers chosen per viewer.
- Livestream: 1 publisher, N subscribers, data channel for chat.

## Tags
[[LiveKit]] [[WebRTC]] [[SFU]] [[RealtimeMedia]]
