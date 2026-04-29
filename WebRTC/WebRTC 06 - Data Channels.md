# 06 — Data Channels

🔑 `RTCDataChannel` is a WebSocket-shaped API on top of SCTP-over-DTLS — bidirectional bytes between peers, with knobs for ordering and reliability.

Source: https://developer.mozilla.org/en-US/docs/Web/API/RTCDataChannel

## Why Not Just WebSocket?
- **P2P** — no server hop; ~half the latency.
- **Unreliable mode** — drop late packets instead of head-of-line blocking.
- **Encrypted** — DTLS by default.
- **Multiplexed** — many channels share one connection.

## Creating
```js
const chan = pc.createDataChannel("chat", {
  ordered: true,
  maxRetransmits: 3
});
chan.onopen = () => chan.send("hi");
chan.onmessage = (e) => console.log(e.data);
```

## Reliability Modes
| `ordered` | `maxRetransmits` | `maxPacketLifeTime` | Behavior |
|---|---|---|---|
| true | unset | unset | **Reliable + ordered** (TCP-like, default) |
| true | 0 | unset | Reliable order, no retry |
| false | N | unset | Up to N retries, any order |
| false | unset | T ms | Retry until T ms elapsed, any order |

⚠️ Set **only one** of `maxRetransmits` / `maxPacketLifeTime` — both throws.

## Message Size & Buffering
Browsers cap at ~64 KiB per `send()` reliably; chunk larger payloads. `bufferedAmount` grows when you outpace the wire — back off on `bufferedamountlow`. Use `binaryType = "arraybuffer"` for binary.

## Use Cases
| Pattern | Why data channel |
|---|---|
| **Chat / presence** | Low latency, P2P |
| **File transfer** | Reliable+ordered, chunked |
| **Game state / cursors** | Unreliable+unordered, prioritize freshness |
| **AI agent control** | Side-channel commands beside audio/video |
| **WebRTC-only apps** | Skip WebSocket entirely |

💡 LiveKit, Discord, and most realtime collab apps push *cursors* and *presence* over data channels because dropping a stale cursor packet beats stalling on retransmission.

🧪 Open the channel from the offerer **before** `createOffer()` — otherwise you'll trigger a renegotiation just to add it.

## Tags
[[WebRTC]] [[RTCDataChannel]]
