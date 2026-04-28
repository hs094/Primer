# 01 — Overview

🔑 LiveKit = open-source WebRTC media server + multi-language SDKs + an Agents framework, packaged as a cloud platform for real-time voice, video, and "physical" AI agents.

Source: https://docs.livekit.io/intro/overview/

## What it is

A full stack for real-time multimodal apps:

- **Media server** — open-source SFU built on WebRTC, routes audio/video/data tracks between participants in a `Room`.
- **SDKs** — clients and servers in many languages talk to the same server over WebRTC + a small signaling protocol.
- **Agents framework** — Python / Node.js runtime for building voice and multimodal AI agents that join rooms as participants.
- **Cloud** — hosted, globally-distributed deployment of the same server, with telephony, recording, and ops baked in. Self-host is also a first-class option.

## Core primitives

- **Room** — the unit of a session. Participants publish/subscribe to tracks.
- **Participant** — a connected client (human, agent, SIP caller, or recorder).
- **Track** — an audio, video, or data stream a participant publishes.
- **Agent** — a server-side participant powered by your code (LLM + STT + TTS + tools).

## SDKs at a glance

| Layer | Languages |
|---|---|
| Client | JavaScript, Swift, Android, Flutter, React Native, Unity |
| Server | Go, Node.js, Python, Ruby |
| UI components | React, Android (Compose), Swift, Flutter |
| Agents | Python, Node.js |

## What it solves

- **WebRTC transport** — NAT traversal, congestion control, simulcast, adaptive bitrate — without you writing it.
- **Media routing** — SFU semantics: subscribe per-track, per-quality, per-participant; scales beyond mesh.
- **Telephony** — SIP integration so PSTN calls land as participants in a room (and vice versa).
- **AI agent plumbing** — turn detection, interruption, tool calling, vision input, all wired into the room model.

## Typical use cases

- Voice AI agents for customer service, intake, triage.
- Phone-bot deployments via SIP (agent answers / dials out).
- Video calls and live streaming with custom UX.
- Multimodal / vision agents that consume the participant's camera feed.
- RAG-backed assistants that join meetings as a participant.

## Where to start

- **Agent Builder** — browser playground, no install.
- **Voice AI quickstart** — minimal Python agent in a room.
- **LiveKit 101** — short video course on the mental model.

## Mental model vs alternatives

- vs **[[Twilio]] Programmable Voice/Video** — LiveKit is WebRTC-first and open-source-core; Twilio is PSTN-first and fully managed. LiveKit Cloud + SIP closes the telephony gap.
- vs raw **WebRTC** — LiveKit gives you an SFU, signaling, auth (JWT room tokens), and SDKs out of the box; you stop reinventing `RTCPeerConnection` glue.
- vs **socket-based chat stacks** — designed for media (codecs, jitter, simulcast), not just messages; data tracks cover the chat case too.

## Tags

[[LiveKit]] [[WebRTC]] [[real-time]] [[voice-AI]] [[agents]] [[SIP]] [[media-server]]
