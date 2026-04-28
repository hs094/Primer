# 03 — Rooms

🔑 `Room` is the session object: it owns participants, fires `RoomEvent`s, and exposes connection + media-routing controls.

Source: https://docs.livekit.io/reference/client-sdk-js/classes/Room.html

## Key Properties
- `localParticipant: LocalParticipant`
- `remoteParticipants: Map<string, RemoteParticipant>` (keyed by identity)
- `activeSpeakers: Participant[]`
- `state: ConnectionState` (`Disconnected`, `Connecting`, `Connected`, `Reconnecting`)
- `name`, `metadata`, `numParticipants`, `numPublishers`
- `isE2EEEnabled`, `options`, `serverInfo`

## Lifecycle Methods
```typescript
await room.prepareConnection(url, token); // pre-warm DNS/ICE
await room.connect(url, token, opts);
await room.disconnect(stopTracks = true);
room.getSid(); // server-assigned room id
```

## Media + Devices
```typescript
await room.startAudio();    // satisfy browser autoplay policies
await room.startVideo();
await room.switchActiveDevice('audioinput', deviceId);
await room.switchActiveDevice('videoinput', deviceId);
await room.switchActiveDevice('audiooutput', speakerId);
```

## Participant Access
```typescript
const peer = room.getParticipantByIdentity('user-42');
```

## Events (`RoomEvent`)
Subscribe via `room.on(RoomEvent.X, handler)`. Common events:
`Connected`, `Disconnected`, `Reconnecting`, `Reconnected`,
`ParticipantConnected`, `ParticipantDisconnected`,
`TrackPublished`, `TrackUnpublished`, `TrackSubscribed`, `TrackUnsubscribed`,
`TrackMuted`, `TrackUnmuted`, `ActiveSpeakersChanged`,
`ConnectionQualityChanged`, `DataReceived`,
`RoomMetadataChanged`, `ParticipantMetadataChanged`, `ParticipantAttributesChanged`.

## Advanced Hooks
```typescript
room.registerRpcMethod('echo', async ({ payload }) => payload);
room.registerTextStreamHandler('chat', async (reader, info) => { /* ... */ });
room.registerByteStreamHandler('files', async (reader, info) => { /* ... */ });
await room.setE2EEEnabled(true); // experimental
```

## Tags
[[LiveKit]] [[WebRTC]] [[Room]] [[Events]]
