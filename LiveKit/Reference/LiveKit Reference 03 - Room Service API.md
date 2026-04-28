# 03 — Room Service API

🔑 Rooms are session containers — created explicitly via `RoomService` or implicitly when the first participant joins, auto-closed when the last leaves.

Source: https://docs.livekit.io/home/server/managing-rooms/

## Concepts

- A room holds participants, tracks, and metadata.
- Auto-creation: configure server defaults so rooms spawn on first join.
- Auto-close: room dies after `empty_timeout` seconds with zero participants.

## Create

```python
from livekit import api
from livekit.protocol.room import CreateRoomRequest

lkapi = api.LiveKitAPI()  # reads LIVEKIT_URL, LIVEKIT_API_KEY, LIVEKIT_API_SECRET

room = await lkapi.room.create_room(
    CreateRoomRequest(
        name="support-1234",
        empty_timeout=10 * 60,    # close after 10 min idle
        max_participants=20,
        metadata='{"topic": "billing"}',
    )
)
```

## List

```python
from livekit.protocol.room import ListRoomsRequest

resp = await lkapi.room.list_rooms(ListRoomsRequest(names=["support-1234"]))
for r in resp.rooms:
    print(r.name, r.num_participants, r.creation_time)
```

## Delete

Disconnects all participants immediately.

```python
from livekit.protocol.room import DeleteRoomRequest
await lkapi.room.delete_room(DeleteRoomRequest(room="support-1234"))
```

## Update room metadata

```python
from livekit.protocol.room import UpdateRoomMetadataRequest
await lkapi.room.update_room_metadata(
    UpdateRoomMetadataRequest(room="support-1234", metadata='{"status":"closed"}')
)
```

## Required grants

`roomCreate`, `roomList`, `roomAdmin` on the signing token.

## Tags
[[LiveKit]] [[Reference]] [[Rooms]] [[API]]
