# VID 01 — Programmable Video

🔑 Programmable Video gives you a `Room` (signalling + media routing) and SDKs to publish/subscribe `Track`s. Backend job: mint short-lived **Access Tokens**. Client job: connect to the room and render tracks.

⚠️ Twilio announced **end-of-life for Programmable Video** (sunset path through 2026). Use it for legacy / migration; for new builds evaluate WebRTC alternatives or Twilio Live before committing.

## Room types

| Type | Participants | Media topology | Recording |
|---|---|---|---|
| `peer-to-peer` (P2P) | 2–10 | mesh | no |
| `group` | up to 50 | SFU (Selective Forwarding Unit) | yes |
| `group-small` | up to 4 | SFU | yes |
| `go` | 2 | SFU | yes (free tier) |

Pick `group` for anything > 2 participants or where you need recording / compositions.

## Concepts

- **Room (`RM…`)** — the meeting. Created on demand or via REST.
- **Participant (`PA…`)** — a connected client identity.
- **Track** — one of `LocalAudioTrack`, `LocalVideoTrack`, `LocalDataTrack`, plus `RemoteAudioTrack` / `RemoteVideoTrack` / `RemoteDataTrack`. Multiple tracks per participant (camera + screen-share).
- **Subscription** — what each participant subscribes to from others (manual, dominant-speaker, switch-off).
- **DataTrack** — small ordered/unreliable messaging channel for in-room state sync.

## Backend — mint an Access Token

```python
from twilio.jwt.access_token import AccessToken
from twilio.jwt.access_token.grants import VideoGrant

def video_token(identity: str, room: str) -> str:
    token = AccessToken(
        os.environ["TWILIO_ACCOUNT_SID"],
        os.environ["TWILIO_API_KEY_SID"],     # SK...
        os.environ["TWILIO_API_KEY_SECRET"],
        identity=identity,
        ttl=3600,
    )
    token.add_grant(VideoGrant(room=room))
    return token.to_jwt()
```

`identity` is whatever string identifies the user in *your* system (must be unique within the room).

## Frontend (JS) — minimal connect

```js
import { connect, createLocalTracks } from "twilio-video";

const tracks = await createLocalTracks({ audio: true, video: { width: 640 } });
const room = await connect(jwtFromBackend, { name: "standup", tracks });

// render local
document.getElementById("self").appendChild(tracks.find(t => t.kind === "video").attach());

// render remotes
room.on("participantConnected", (p) => {
  p.on("trackSubscribed", (t) => {
    document.getElementById("remotes").appendChild(t.attach());
  });
});
```

Disconnect:

```js
room.disconnect();
tracks.forEach(t => t.stop());
```

## REST API — create / inspect rooms

```python
room = client.video.v1.rooms.create(
    unique_name="standup",
    type="group",
    record_participants_on_connect=True,
    media_region="ie1",                    # closest media region for participants
    status_callback="https://app.example.com/twilio/room",
    status_callback_method="POST",
)

# end the room
client.video.v1.rooms(room.sid).update(status="completed")

# list participants
for p in client.video.v1.rooms(room.sid).participants.list():
    print(p.identity, p.status)
```

`type` is also `group-small`, `peer-to-peer`, or `go`.

`media_region`: `us1`, `us2`, `ie1`, `de1`, `au1`, `jp1`, `sg1`, `br1`. Pick closest to most participants.

## Recording & composition

`record_participants_on_connect=true` records each participant's tracks individually (raw recordings, `RT…`). To produce a single playable file:

```python
comp = client.video.v1.compositions.create(
    room_sid=room.sid,
    audio_sources=["*"],
    video_layout={
        "grid": {
            "video_sources": ["*"],
        }
    },
    format="mp4",
    status_callback="https://app.example.com/twilio/composition",
)
```

Compositions are async — you get a webhook with a `MediaUrl` once rendered. Recordings + compositions live in Twilio storage by default; route to **AWS S3** with **External S3 Storage** for own-bucket retention.

## Bandwidth & quality controls

- **Track switch-off** — SFU stops sending video tracks the user isn't displaying (e.g., off-screen tiles). Saves egress, automatic on `group` rooms.
- **Bandwidth profile** — limit max video bitrate / count of subscribed tracks. Set via SDK on `connect(...)`.
- **Network Quality API** — per-participant 0–5 score on each leg, surfaces in SDK events.
- **Dominant Speaker** — Twilio designates the loudest participant; SDK fires `dominantSpeakerChanged`.

## Push / camera permissions

Browsers gate `getUserMedia`. Always request audio/video in a user gesture handler. iOS Safari requires HTTPS + secure context.

## Sunset / migration

For new builds:

- Voice + Video AI agents → **Voice ConversationRelay + Media Streams**, plus a video provider (LiveKit, Daily, Vonage).
- Generic WebRTC → consider building on `mediasoup`, LiveKit, or Daily — Twilio's roadmap is moving away from Programmable Video.

Keep Programmable Video docs for understanding existing apps; budget a migration project before peak-2026.
