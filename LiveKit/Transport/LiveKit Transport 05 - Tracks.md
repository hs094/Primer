# 05 — Tracks

🔑 Tracks are the unit of media; `TrackPublication` is the server-registered metadata, `Track` is the live stream once subscribed.

Source: https://docs.livekit.io/home/client/tracks/

## Kinds
- `Track.Kind.Audio` — mic, synthesized speech, music, screen-share audio.
- `Track.Kind.Video` — camera, screen, synthetic frames from agents.
- Data is handled separately (data packets / streams / RPC) — not as `Track`.

## Sources
`Track.Source`:
- `Microphone`
- `Camera`
- `ScreenShare`
- `ScreenShareAudio`
- `Unknown` (custom)

A participant typically holds one track per source but may publish many.

## Publication vs Track
```typescript
// Publication exists on every participant once announced
const pub = remote.getTrackPublication(Track.Source.Microphone);
pub.trackSid;       // server id
pub.isMuted;
pub.isSubscribed;
pub.track;          // null until subscribed
```
`Track` becomes non-null after `TrackSubscribed`. It exposes `attach()`, `detach()`, `mediaStreamTrack`, and source-specific helpers.

## Capabilities
- Camera + mic capture with device selection.
- Screen share (browser `getDisplayMedia`, native APIs on iOS/Android).
- Adaptive streaming + simulcast layers chosen by the SFU per subscriber.
- Raw frame access for processing (`AudioFrame`, `VideoFrame`).
- Built-in noise/echo cancellation (Krisp on cloud).
- Codec config (Opus, VP8, VP9, H.264, AV1).
- Stream import/export (ingress/egress) for RTMP, SIP, HLS.

## Example: enumerate tracks
```python
for sid, pub in remote_participant.track_publications.items():
    print(f"{sid} kind={pub.kind} source={pub.source} subscribed={pub.subscribed}")
```

## Tags
[[LiveKit]] [[Tracks]] [[WebRTC]] [[Media]]
