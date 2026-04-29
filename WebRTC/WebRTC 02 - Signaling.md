# 02 — Signaling

🔑 [[WebRTC]] deliberately defines **no signaling protocol**. Two peers must exchange SDP and ICE candidates somehow — that "somehow" is your problem.

Source: https://developer.mozilla.org/en-US/docs/Web/API/WebRTC_API/Signaling_and_video_calling

## Why It's Out Of Scope
Apps already have a backend, an auth model, and a transport (WebSocket, HTTP, MQTT, SIP). WebRTC reuses whatever you have rather than mandating a 13th messaging system.

## What You Must Exchange
| Payload | When | Direction |
|---|---|---|
| **SDP offer** | Caller starts negotiation | A → B |
| **SDP answer** | Callee accepts | B → A |
| **ICE candidates** | Continuously as discovered | both ways |
| **Renegotiation offers** | Tracks added/removed | either side |

## Typical Stack
- **WebSocket** — most common; full-duplex, low overhead.
- **HTTP long-poll / SSE** — fine for low-rate apps.
- **SIP over WebSocket** — for telephony interop.
- **Matrix, XMPP, MQTT** — when you already speak them.

## Minimal Flow
1. A: `createOffer()` → `setLocalDescription(offer)` → send offer over signaling.
2. B: `setRemoteDescription(offer)` → `createAnswer()` → `setLocalDescription(answer)` → send answer.
3. Both sides emit `icecandidate` events → forward each candidate over signaling → other side calls `addIceCandidate`.

💡 **Trickle ICE** — send candidates as they arrive instead of waiting for gathering to finish. Faster connect time. Default in modern stacks.

⚠️ Signaling channel must outlive negotiation — you'll need it again for renegotiation, ICE restarts, and disconnect notifications.

🧪 [[LiveKit]], Daily, Twilio etc. all wrap this — your client talks their SDK, and they speak SDP/ICE under the hood.

## Tags
[[WebRTC]] [[SDP]] [[ICE]] [[LiveKit]]
