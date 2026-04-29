# 05 — RTCPeerConnection

🔑 `RTCPeerConnection` is the single object that wraps ICE, DTLS, SRTP, RTP, and SCTP. Everything else in [[WebRTC]] hangs off it.

Source: https://developer.mozilla.org/en-US/docs/Web/API/RTCPeerConnection

## Lifecycle
```js
const pc = new RTCPeerConnection({ iceServers });
stream.getTracks().forEach(t => pc.addTrack(t, stream));
const offer = await pc.createOffer();
await pc.setLocalDescription(offer);
// send offer.sdp over signaling, get answer back
await pc.setRemoteDescription(answer);
```

## Key Methods
| Method | Purpose |
|---|---|
| `addTrack(track, stream)` | Send a local audio/video track |
| `createOffer()` / `createAnswer()` | Build [[SDP]] |
| `setLocalDescription` / `setRemoteDescription` | Apply SDP |
| `addIceCandidate(c)` | Feed remote candidates |
| `createDataChannel(label)` | Open an [[RTCDataChannel]] |
| `getStats()` | Pull metrics (jitter, RTT, loss) |
| `close()` | Tear down |

## Key Events
- `negotiationneeded` — track added/removed; rebuild offer.
- `icecandidate` — forward `event.candidate` over signaling.
- `track` — remote media arrived; attach `event.streams[0]` to a `<video>`.
- `datachannel` — remote opened a channel.
- `connectionstatechange` — `connecting → connected → disconnected → failed → closed`.

## Perfect Negotiation
💡 Pattern from the spec for symmetric apps where either side may renegotiate:
- Each side has a `polite` boolean (one true, one false).
- On `negotiationneeded` → make an offer.
- If you receive an offer while already offering: the **polite** peer rolls back, the **impolite** peer ignores.
- Eliminates glare without a turn-taking protocol.

## Transceivers
⚠️ Every m-section has an `RTCRtpTransceiver` with a sender + receiver. Direction is `sendrecv | sendonly | recvonly | inactive`. Modern code prefers `addTransceiver()` over `addTrack()` for explicit control of slot allocation.

## Stats & Renegotiation
🧪 `pc.getStats()` returns a `RTCStatsReport` map — feed it to a dashboard. Renegotiate by repeating createOffer/answer; SDP changes are diffed automatically.

## Tags
[[WebRTC]] [[SDP]] [[ICE]]
