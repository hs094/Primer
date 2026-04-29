# 07 — Codecs

🔑 [[WebRTC]] mandates a small set of codecs every browser must support — so any two peers can always find a match.

Source: https://developer.mozilla.org/en-US/docs/Web/Media/Guides/Formats/WebRTC_codecs

## Audio (MTI = Mandatory To Implement)
| Codec | Notes |
|---|---|
| **[[Opus]]** | Default. 6–510 kbps, 8–48 kHz, NB/WB/SWB/FB. In-band FEC, DTX. |
| **G.711** (PCMU/PCMA) | 64 kbps, 8 kHz. Telephony interop only. |
| G.722, iLBC, iSAC | Optional / legacy. |

💡 [[Opus]] is the right answer for almost everything — it adapts from podcast-quality music to robust low-bandwidth speech in one codec.

## Video (MTI)
| Codec | Notes |
|---|---|
| **VP8** | Universal, royalty-free. Baseline target. |
| **VP9** | ~30% better than VP8, supports SVC. |
| **H.264** | Hardware accel almost everywhere; safe for mobile battery. |
| **AV1** | Best compression, expensive to encode in software. Hardware support growing. |
| **H.265 / HEVC** | Patent-encumbered; Safari only. |

## Simulcast
⚠️ Send the **same video** at multiple resolutions/bitrates simultaneously (e.g., 1080p + 540p + 180p). The [[SFU]] then forwards whichever layer each receiver can handle.
- Supported on VP8, H.264, VP9, AV1.
- Configured via `RTCRtpSender.setParameters({ encodings: [{rid:"f"}, {rid:"h"}, {rid:"q"}] })`.

## SVC (Scalable Video Coding)
🧪 One bitstream encodes multiple temporal/spatial layers; SFU drops layers per-receiver. More efficient than simulcast (single encode), but only VP9/AV1 do it well.
- **Temporal** — drop frames to reduce framerate.
- **Spatial** — drop layers to reduce resolution.
- LiveKit defaults to VP9 SVC for video.

## Negotiation
Codec list lives in SDP `a=rtpmap` / `a=fmtp` lines. Order signals preference. Use `RTCRtpTransceiver.setCodecPreferences()` to influence — never munge SDP strings.

## Tags
[[WebRTC]] [[Opus]] [[SDP]] [[SFU]]
