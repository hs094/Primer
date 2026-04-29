# 03 — SDP

🔑 [[SDP]] (Session Description Protocol, RFC 8866) is the plain-text format that describes "what media this peer wants to send/receive and how to decode it."

Source: https://developer.mozilla.org/en-US/docs/Web/API/RTCSessionDescription

## Anatomy
An SDP blob is line-oriented `key=value`. Two scopes:
- **Session-level** — `v=`, `o=`, `s=`, `t=`, `c=`.
- **Media-level** — one `m=` block per stream (audio, video, data), followed by `a=` attributes.

## The Important Lines
| Line | Meaning |
|---|---|
| `m=audio 9 UDP/TLS/RTP/SAVPF 111 63` | Audio section, payload types 111, 63 |
| `m=video 9 UDP/TLS/RTP/SAVPF 96 97` | Video section |
| `m=application 9 UDP/DTLS/SCTP webrtc-datachannel` | Data channel |
| `a=rtpmap:111 opus/48000/2` | PT 111 = [[Opus]], 48 kHz, stereo |
| `a=fmtp:111 minptime=10;useinbandfec=1` | Codec-specific params |
| `a=rtcp-fb:96 nack pli` | Feedback: NACK + Picture Loss Indication |
| `a=sendrecv` / `a=recvonly` / `a=sendonly` | Direction |
| `a=ice-ufrag` / `a=ice-pwd` | ICE creds |
| `a=fingerprint:sha-256 ...` | DTLS cert fingerprint |
| `a=mid:0` | Media ID, used by BUNDLE |
| `a=group:BUNDLE 0 1 2` | Multiplex all m-sections on one transport |

## Offer / Answer
💡 The **offer** lists everything the offerer *can* do; the **answer** picks the intersection. Result is the negotiated set of codecs, directions, and transport params.

## BUNDLE & RTCP-MUX
⚠️ Without BUNDLE, each m-section needs its own ICE/DTLS handshake. With BUNDLE (default), audio + video + data all share one UDP socket. Massively faster connect.

## Don't Hand-Edit
🧪 SDP munging used to be common (force codec, cap bitrate). Modern API: `RTCRtpSender.setParameters()` and codec preferences via `setCodecPreferences()`. Mungers break across browser updates.

## Tags
[[SDP]] [[WebRTC]] [[Opus]]
