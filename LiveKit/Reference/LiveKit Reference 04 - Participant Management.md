# 04 — Participant Management

🔑 `RoomService` exposes participant CRUD, permission mutation, metadata sync, mute, and forced disconnect — all gated by `roomAdmin`.

Source: https://docs.livekit.io/home/server/managing-participants/

## Participant kinds

`STANDARD`, `AGENT`, `SIP`, `EGRESS`, `INGRESS` — distinguishable via the `kind` field on `ParticipantInfo`.

## List participants

```python
from livekit.protocol.room import ListParticipantsRequest

resp = await lkapi.room.list_participants(
    ListParticipantsRequest(room="support-1234")
)
for p in resp.participants:
    print(p.identity, p.kind, p.state)
```

## Get one

```python
from livekit.protocol.room import RoomParticipantIdentity
p = await lkapi.room.get_participant(
    RoomParticipantIdentity(room="support-1234", identity="caller-99")
)
```

## Update permissions

Revoking `can_publish` auto-unpublishes existing tracks.

```python
from livekit.protocol.room import UpdateParticipantRequest
from livekit.protocol.models import ParticipantPermission

await lkapi.room.update_participant(
    UpdateParticipantRequest(
        room="support-1234",
        identity="caller-99",
        permission=ParticipantPermission(
            can_publish=False,
            can_subscribe=True,
            can_publish_data=True,
        ),
    )
)
```

## Update metadata

Synced to all connected clients.

```python
await lkapi.room.update_participant(
    UpdateParticipantRequest(
        room="support-1234",
        identity="caller-99",
        metadata='{"role": "guest"}',
    )
)
```

## Mute a track

```python
from livekit.protocol.room import MuteRoomTrackRequest
await lkapi.room.mute_published_track(
    MuteRoomTrackRequest(
        room="support-1234",
        identity="caller-99",
        track_sid="TR_xxx",
        muted=True,
    )
)
```

## Remove participant

```python
from livekit.protocol.room import RoomParticipantIdentity
await lkapi.room.remove_participant(
    RoomParticipantIdentity(room="support-1234", identity="caller-99")
)
```

## Forward / move (Cloud only)

Forward a participant's tracks into another room, or move them outright. Requires `destination_room` grant on the token.

## Tags
[[LiveKit]] [[Reference]] [[Participants]] [[API]]
