# 07 — Subscribing to Media

🔑 Auto-subscribe is on by default; handle `TrackSubscribed` to attach the materialized `Track` to a UI element.

Source: https://docs.livekit.io/home/client/tracks/subscribe/

## Default Behavior
`autoSubscribe: true` — server pushes new tracks to every participant as they appear. Disable for selective subscription.

```typescript
const room = new Room();
await room.connect(url, token, { autoSubscribe: false });
```

## TrackSubscribed Callback
Fires on both `Room` and the originating `RemoteParticipant`:
```typescript
import { RoomEvent, RemoteTrack, RemoteTrackPublication, RemoteParticipant } from 'livekit-client';

room.on(RoomEvent.TrackSubscribed, (
  track: RemoteTrack,
  publication: RemoteTrackPublication,
  participant: RemoteParticipant,
) => {
  if (track.kind === 'audio' || track.kind === 'video') {
    const el = track.attach();
    document.getElementById('media')!.appendChild(el);
  }
});

room.on(RoomEvent.TrackUnsubscribed, (track) => {
  track.detach().forEach((el) => el.remove());
});
```

## Manual Subscribe
```typescript
publication.setSubscribed(true);
publication.setSubscribed(false);
```

## Volume + Playback
```typescript
audioTrack.setVolume(0.5);          // 0.0 – 1.0
await room.startAudio();             // unblock autoplay restrictions
```

## Platform Renderers
- React: `<VideoTrack track={publication.track} />`, `<AudioTrack />`
- Swift: `VideoView`
- Android: `VideoTrackRenderer` (Compose) or `SurfaceViewRenderer`
- Flutter: `VideoTrackRenderer`

## Server-side Subscription Control
Node.js / Go server SDKs:
```typescript
await roomService.updateSubscriptions(roomName, identity, [trackSid], true);
```
Use to gate consumption from a backend (recording, moderation, billing).

## Python Subscribe
```python
@room.on("track_subscribed")
def on_subscribed(track: rtc.Track, pub: rtc.RemoteTrackPublication, p: rtc.RemoteParticipant):
    if isinstance(track, rtc.RemoteAudioTrack):
        async for frame in rtc.AudioStream(track):
            process(frame)
```

## Tags
[[LiveKit]] [[Subscribe]] [[Tracks]] [[Media]]
