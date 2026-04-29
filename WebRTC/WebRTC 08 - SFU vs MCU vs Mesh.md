# 08 — SFU vs MCU vs Mesh

🔑 [[WebRTC]] is peer-to-peer at the protocol level, but only one of the three multi-party topologies actually scales. That's the [[SFU]].

Source: https://webrtc.org/getting-started/media-devices + LiveKit docs

## Mesh (Full P2P)
Every participant opens an `RTCPeerConnection` to every other participant. N participants → N·(N−1)/2 connections; each peer uploads (N−1) copies of its own video.

| Pros | Cons |
|---|---|
| No server media cost | Upload bandwidth dies past ~4 peers |
| End-to-end encrypted by default | No recording, no broadcast, no mobile battery |

✅ Use for: 1:1 calls, demos, ≤4-person prototypes.

## SFU (Selective Forwarding Unit)
Each client sends **one** stream up to a server; the server **forwards** copies to every other client without decoding.

⚠️ Pivotal property: server doesn't transcode → no GPU, no quality loss, low CPU. Just routing.
- Combines beautifully with **simulcast / SVC** — server picks the right layer per receiver.
- Scales to thousands of receivers per publisher.

| Implementations | |
|---|---|
| **[[LiveKit]]** | Go, opinionated, end-to-end product |
| **mediasoup** | Node.js library, build-your-own |
| **Janus** | C, plugin-based |
| **Pion / ion-sfu** | Go, modular |
| **Jitsi Videobridge** | Java, Jitsi Meet's engine |

✅ Use for: anything multi-party, conferencing, livestream, AI rooms.

## MCU (Multipoint Control Unit)
Server **decodes** every input, composes them into one mixed stream, **re-encodes**, and sends back.

| Pros | Cons |
|---|---|
| Client decodes 1 stream — cheap devices love it | Massive server CPU/GPU |
| Composited recording trivial; SIP/PSTN trunks | Quality loss, latency, $$$, no simulcast |

✅ Use for: legacy telephony bridges, very low-end clients, single-stream broadcast outputs.

## Rule of Thumb
💡 Mesh ≤4 → SFU for everything else → MCU only when you absolutely must mix.

🧪 [[LiveKit]] is an SFU at heart; agents join as participants; recording/composition is a separate egress service that *acts like* an MCU only when you ask.

## Tags
[[SFU]] [[WebRTC]] [[LiveKit]]
