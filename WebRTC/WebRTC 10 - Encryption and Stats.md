# 10 — Encryption & Stats

🔑 [[WebRTC]] is **always** encrypted — DTLS-SRTP is mandatory by spec. There is no "plaintext" mode. Observability is via `getStats()` and `chrome://webrtc-internals`.

Source: https://developer.mozilla.org/en-US/docs/Web/API/WebRTC_API/Protocols

## DTLS-SRTP
| Layer | Job |
|---|---|
| **DTLS** | TLS-over-UDP handshake; mutual auth via self-signed cert fingerprints exchanged in [[SDP]] (`a=fingerprint`) |
| **SRTP** | Symmetric keys derived from DTLS protect every RTP/RTCP packet (AES-GCM or AES-CM + HMAC-SHA1) |
| **SCTP-over-DTLS** | Same DTLS tunnel encrypts data channels |

⚠️ Self-signed certs alone don't prove peer identity — that's signaling's job. If signaling is hijacked, an attacker can MITM. Mitigations: identity assertions, or trust the SFU.

## E2E Encryption (Insertable Streams)
🧪 For true end-to-end across an [[SFU]], use **Encoded Transform** / **Insertable Streams** — encrypt frames in a Worker before they hit the encoder, decrypt after the decoder. SFU sees ciphertext only.

## getStats()
```js
const report = await pc.getStats();
report.forEach(s => console.log(s.type, s));
```
Returns an `RTCStatsReport` (a Map). Key report types:

| `type` | Useful fields |
|---|---|
| `inbound-rtp` | `packetsLost`, `jitter`, `framesDecoded`, `framesDropped`, `bytesReceived` |
| `outbound-rtp` | `packetsSent`, `bytesSent`, `targetBitrate`, `framesPerSecond`, `qualityLimitationReason` |
| `remote-inbound-rtp` | `roundTripTime`, peer's view of *your* loss/jitter |
| `candidate-pair` | `currentRoundTripTime`, `state`, which pair is nominated |
| `transport` | `dtlsState`, `selectedCandidatePairId`, bytes |

💡 Sample every 1–2s, diff counters, push to your metrics pipeline.

## Common Diagnoses
- **High jitter, low loss** → network congestion → drop bitrate / spatial layer.
- **packetsLost climbing** → switch to TURN, enable FEC, drop framerate.
- **qualityLimitationReason: "cpu"** → encoder can't keep up → lower resolution.
- **iceConnectionState: "failed"** → trigger ICE restart.

## Debugging
🧪 `chrome://webrtc-internals` — every PC, every stat, live graphs. First stop for any "the call dropped" ticket. Firefox: `about:webrtc`.

## Tags
[[WebRTC]] [[SDP]]
