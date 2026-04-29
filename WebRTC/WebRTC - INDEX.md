# WebRTC — INDEX

🔑 [[WebRTC]] fundamentals: how browsers and native apps move audio, video, and bytes peer-to-peer in realtime.

Sources: https://webrtc.org/ + https://developer.mozilla.org/en-US/docs/Web/API/WebRTC_API

## Notes
1. [[WebRTC 01 - Overview]] — what WebRTC is, who backs it, what it gives you (and doesn't).
2. [[WebRTC 02 - Signaling]] — why signaling is out of scope; SDP + ICE exchange over your transport of choice.
3. [[WebRTC 03 - SDP]] — Session Description Protocol, m-sections, a-attributes, BUNDLE.
4. [[WebRTC 04 - ICE STUN TURN]] — NAT traversal, candidate types, trickle ICE, when TURN kicks in.
5. [[WebRTC 05 - Peer Connection API]] — `RTCPeerConnection` lifecycle, transceivers, perfect negotiation.
6. [[WebRTC 06 - Data Channels]] — `RTCDataChannel`, ordered/unordered, reliable/unreliable, use cases.
7. [[WebRTC 07 - Codecs]] — Opus, VP8/VP9/H.264/AV1, simulcast, SVC.
8. [[WebRTC 08 - SFU vs MCU vs Mesh]] — multi-party topologies; why SFUs win.
9. [[WebRTC 09 - getUserMedia]] — capturing camera, mic, screen; constraints, permissions.
10. [[WebRTC 10 - Encryption and Stats]] — DTLS-SRTP, `getStats()`, `chrome://webrtc-internals`.

## Concept Map
| Layer | Notes |
|---|---|
| Capture | [[WebRTC 09 - getUserMedia]] |
| Negotiation | [[WebRTC 02 - Signaling]] · [[WebRTC 03 - SDP]] |
| Connectivity | [[WebRTC 04 - ICE STUN TURN]] |
| Transport | [[WebRTC 05 - Peer Connection API]] · [[WebRTC 06 - Data Channels]] |
| Media | [[WebRTC 07 - Codecs]] |
| Topology | [[WebRTC 08 - SFU vs MCU vs Mesh]] |
| Security & Ops | [[WebRTC 10 - Encryption and Stats]] |

## Cross-Pack
💡 [[LiveKit]] is the production stack built on these primitives — it's an [[SFU]] with auth, agents, telephony, and SDKs around the bare WebRTC API.
- [[LiveKit Intro 02 - Basics]] — Rooms / Participants / Tracks on top of WebRTC.
- [[LiveKit 01 - Overview]] — where this pack picks up.

## Tags
[[WebRTC]] [[SDP]] [[ICE]] [[STUN]] [[TURN]] [[SFU]] [[Opus]] [[LiveKit]]
