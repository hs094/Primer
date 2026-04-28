# 02 — Basics

🔑 LiveKit's mental model is three primitives: Rooms, Participants, and Tracks — agents are just participants.

Source: https://docs.livekit.io/intro/basics/

## Core Architecture
| Primitive | Role |
|---|---|
| **Room** | Virtual space where communication happens |
| **Participant** | Entity (user or agent) joined to a room |
| **Track** | Media stream (audio, video, data) flowing between participants |

## What This Buys You
- **AI Agent integration** — Agents framework lets agents join rooms as participants, process realtime media, interact via voice/text/vision.
- **LiveKit Cloud** — managed, global mesh; agent hosting, managed inference, native telephony, observability.
- **LiveKit CLI (`lk`)** — project management, scaffolding, agent deploy.

## Connection Model
Apps connect via:
1. Access token (encodes room, identity, permissions)
2. WebRTC connection over `wsUrl`
3. Platform SDK (Web/iOS/Android/Flutter/Unity/backend)

Same infrastructure carries both human users and AI agents.

## Tags
[[LiveKit]] [[Rooms]] [[Participants]] [[Tracks]] [[WebRTC]]
