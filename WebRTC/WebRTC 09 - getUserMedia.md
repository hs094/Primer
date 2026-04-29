# 09 — getUserMedia

🔑 `navigator.mediaDevices.getUserMedia()` is the gateway: it asks the user for the camera/mic and hands back a `MediaStream` you can attach to `<video>` or feed into `RTCPeerConnection.addTrack()`.

Source: https://developer.mozilla.org/en-US/docs/Web/API/MediaDevices/getUserMedia

## Basic Call
```js
const stream = await navigator.mediaDevices.getUserMedia({
  audio: true,
  video: { width: 1280, height: 720, facingMode: "user" }
});
videoEl.srcObject = stream;
```

## Constraints
| Field | Examples |
|---|---|
| `video.width` / `height` | exact `{ exact: 1920 }`, ideal `{ ideal: 1280 }` |
| `video.frameRate` | `{ ideal: 30, max: 60 }` |
| `video.facingMode` | `"user"`, `"environment"` |
| `video.deviceId` | `{ exact: id }` from `enumerateDevices()` |
| `audio.echoCancellation` | `true` (default) |
| `audio.noiseSuppression`, `autoGainControl` | browser DSP |
| `audio.sampleRate`, `channelCount` | usually leave alone |

⚠️ `{ exact: ... }` will **reject** if unmet. Prefer `ideal` and let the browser pick.

## Permissions
- Requires **secure context** (HTTPS or localhost).
- User must grant per-origin; Permissions API: `navigator.permissions.query({ name: "camera" })`.
- Denial → `NotAllowedError`. No device → `NotFoundError`. Hardware busy → `NotReadableError`.

## MediaStream / MediaStreamTrack
A `MediaStream` is a bag of `MediaStreamTrack`s. `track.stop()` releases hardware (camera light off); `track.applyConstraints({...})` retunes live; `track.enabled = false` mutes locally without releasing.

## Device Enumeration
```js
const devices = await navigator.mediaDevices.enumerateDevices();
// devices[i].kind = "videoinput" | "audioinput" | "audiooutput"
```
💡 Labels are empty until the user has granted permission once. Request a stream first, then enumerate.

## Screen Capture
🧪 `navigator.mediaDevices.getDisplayMedia({ video: true, audio: true })` — separate API, separate prompt. User picks a window/tab/screen. Audio capture is OS-dependent (works on Chrome+Windows tab capture; spotty elsewhere).

## Tags
[[WebRTC]] [[getUserMedia]]
