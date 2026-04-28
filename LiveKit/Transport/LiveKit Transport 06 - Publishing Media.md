# 06 — Publishing Media

🔑 Toggle camera/mic with one-line setters on `LocalParticipant`; backend agents publish raw `AudioSource`/`VideoSource` frames.

Source: https://docs.livekit.io/home/client/tracks/publish/

## Browser One-Liners
```typescript
await room.localParticipant.setCameraEnabled(true);
await room.localParticipant.setMicrophoneEnabled(true);
await room.localParticipant.setScreenShareEnabled(true);
```
First call triggers the browser permission prompt. Returns the resulting `LocalTrackPublication`.

## Native Permissions
- iOS `Info.plist`: `NSCameraUsageDescription`, `NSMicrophoneUsageDescription`. Background audio needs the "Audio, AirPlay, and Picture in Picture" capability.
- Android `AndroidManifest.xml`: `CAMERA`, `RECORD_AUDIO`; request at runtime.

## Mute / Unmute
```typescript
const pub = room.localParticipant.getTrackPublication(Track.Source.Microphone);
await pub.mute();
await pub.unmute();
```
Fires `TrackMuted` / `TrackUnmuted` on every participant.

## Subscription Permissions
```typescript
room.localParticipant.setTrackSubscriptionPermissions(false, [
  { participantIdentity: 'agent-1', allowAll: true },
]);
```
First arg = `allParticipantsAllowed`; second = explicit per-identity grants.

## Publishing from a Backend (Python agents)
Audio:
```python
from livekit import rtc

SAMPLE_RATE = 48000
NUM_CHANNELS = 1

source = rtc.AudioSource(SAMPLE_RATE, NUM_CHANNELS)
track = rtc.LocalAudioTrack.create_audio_track("example-track", source)
options = rtc.TrackPublishOptions(source=rtc.TrackSource.SOURCE_MICROPHONE)
publication = await ctx.agent.publish_track(track, options)

await source.capture_frame(audio_frame)  # awaitable; respects sample clock
```

Video:
```python
WIDTH, HEIGHT = 1280, 720

source = rtc.VideoSource(WIDTH, HEIGHT)
track = rtc.LocalVideoTrack.create_video_track("example-track", source)
options = rtc.TrackPublishOptions(source=rtc.TrackSource.SOURCE_CAMERA)
publication = await ctx.agent.publish_track(track, options)

source.capture_frame(video_frame)  # non-blocking
```

## A/V Sync (Python)
```python
from livekit.agents.utils import AVSynchronizer

sync = AVSynchronizer(audio_source, video_source, video_fps=30, audio_sample_rate=SAMPLE_RATE)
await sync.push_audio(audio_frame)
await sync.push_video(video_frame)
```
Aligns first frames, then drives both streams off a shared clock.

## Tags
[[LiveKit]] [[Publishing]] [[Agents]] [[Media]]
