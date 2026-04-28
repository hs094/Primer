# 04 — Participants

🔑 `Participant` is the base class; `LocalParticipant` publishes, `RemoteParticipant` is observed. State syncs via `attributes` (k/v) and `metadata` (opaque string).

Source: https://docs.livekit.io/reference/client-sdk-js/classes/Participant.html

## Identity + State
- `identity: string` — client-chosen, encoded in JWT, stable per session.
- `sid: string` — server-assigned id.
- `name?: string`, `metadata?: string`, `attributes: Readonly<Record<string,string>>`
- `permissions?: ParticipantPermission`
- `audioLevel: number` (0–1.0), `isSpeaking: boolean`, `lastSpokeAt?: Date`
- Accessors: `isActive`, `isAgent`, `isLocal`, `isEncrypted`, `connectionQuality`, `joinedAt`, `isCameraEnabled`, `isMicrophoneEnabled`, `isScreenShareEnabled`.

## Track Maps
- `trackPublications: Map<string, TrackPublication>` (by track sid)
- `audioTrackPublications`, `videoTrackPublications`
- `getTrackPublication(source)` — `Track.Source.Camera | Microphone | ScreenShare | ScreenShareAudio`
- `getTrackPublicationByName(name)`
- `getTrackPublications()` returns array.

## Awaiting Activation
```typescript
await participant.waitUntilActive(); // resolves once media path is up
```

## LocalParticipant Mutations
```typescript
await room.localParticipant.setMetadata('{"role":"host"}');
await room.localParticipant.setAttributes({ status: 'busy', lang: 'en' });
await room.localParticipant.setName('Alice');
```

## Python (agents)
```python
local = room.local_participant
await local.set_attributes({"status": "thinking"})
await local.set_metadata(f"build={build_id}")
```

## Reacting to Updates
```typescript
room.on(RoomEvent.ParticipantAttributesChanged, (changed, participant) => {
  console.log(f := participant.identity, changed);
});
room.on(RoomEvent.ParticipantMetadataChanged, (prev, participant) => {});
```

## Permissions Field
`ParticipantPermission` carries `canPublish`, `canSubscribe`, `canPublishData`, `canPublishSources`, `hidden`, `recorder` — set in the JWT `VideoGrant`, enforced server-side.

## Tags
[[LiveKit]] [[Participants]] [[State]] [[Attributes]]
