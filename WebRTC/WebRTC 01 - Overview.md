# 01 — Overview

🔑 [[WebRTC]] is an open standard for realtime audio, video, and arbitrary data between peers — directly in browsers and native apps, with no plugins.

Source: https://webrtc.org/ + https://developer.mozilla.org/en-US/docs/Web/API/WebRTC_API

## What It Is
| Aspect | Detail |
|---|---|
| **Standards** | W3C (JS APIs) + IETF (wire protocols) |
| **Transport** | UDP-first, P2P where possible, encrypted end-to-end |
| **Media** | Audio, video, generic data channels |
| **Reach** | All major browsers + native libs (iOS, Android, desktop) |

## Backed By
Apple, Google, Microsoft, Mozilla — open-source reference implementation (`libwebrtc`) used by Chrome, Safari, Firefox, Edge.

## What WebRTC Gives You
- **Capture** — `getUserMedia` for camera/mic, `getDisplayMedia` for screen.
- **Transport** — `RTCPeerConnection` negotiates a secure media path through NATs.
- **Data** — `RTCDataChannel` for low-latency bidirectional bytes (chat, game state, file transfer).
- **Encryption** — DTLS-SRTP mandatory; you cannot turn it off.

## What WebRTC Does *Not* Give You
⚠️ No signaling, no rooms, no auth, no recording, no scaling beyond a handful of peers. Apps build that on top — or use an SFU like [[LiveKit]], mediasoup, Janus.

## Use Cases
- Video conferencing (Meet, Teams, Zoom Web)
- Voice calling, telephony (WebRTC ↔ SIP gateways)
- Live streaming (WHIP/WHEP ingest)
- Realtime AI agents (voice bots, vision pipelines)
- P2P file transfer, multiplayer games, remote desktop

## Tags
[[WebRTC]] [[LiveKit]] [[SDP]] [[ICE]]
