# 09 — Token Generation

🔑 LiveKit auth is a JWT signed with your API secret carrying a `VideoGrants` payload — minted server-side, consumed by clients to join rooms.

Source: https://docs.livekit.io/home/get-started/authentication/

## Token sources (client-side abstraction)

1. **Sandbox** — Cloud-hosted dev token server, never production.
2. **Endpoint** — your backend exposes a route returning `{ accessToken, serverUrl }`.
3. **Custom** — async function in client code.
4. **Literal** — pre-generated token string.

## Mint a join token (Python)

```python
from datetime import timedelta
from livekit import api

token = (
    api.AccessToken(api_key="API_KEY", api_secret="API_SECRET")
    .with_identity("user-42")
    .with_name("Alice")
    .with_metadata('{"role": "host"}')
    .with_ttl(timedelta(hours=6))
    .with_grants(
        api.VideoGrants(
            room_join=True,
            room="support-1234",
            can_publish=True,
            can_subscribe=True,
            can_publish_data=True,
        )
    )
    .to_jwt()
)
```

## VideoGrants reference

| Grant | Purpose |
|---|---|
| `room_join` | allow joining `room` |
| `room` | room name (with `room_join`) |
| `room_create` | create rooms via server API |
| `room_list` | list rooms |
| `room_admin` | manage rooms/participants |
| `room_record` | start egress |
| `can_publish` | publish tracks |
| `can_subscribe` | subscribe to tracks |
| `can_publish_data` | publish data messages |
| `can_publish_sources` | restrict to `["camera","microphone","screen_share"]` |
| `can_update_own_metadata` | self-mutate metadata |
| `hidden` | invisible participant |
| `recorder` | mark as recorder bot |
| `agent` | mark as agent participant |

## Backend endpoint pattern (FastAPI)

```python
from datetime import timedelta
from fastapi import FastAPI, Depends
from pydantic import BaseModel
from livekit import api

app = FastAPI()

class TokenReq(BaseModel):
    room: str
    identity: str

class TokenResp(BaseModel):
    access_token: str
    server_url: str

@app.post("/livekit/token", response_model=TokenResp)
async def mint(body: TokenReq) -> TokenResp:
    jwt = (
        api.AccessToken()  # reads LIVEKIT_API_KEY / LIVEKIT_API_SECRET
        .with_identity(body.identity)
        .with_ttl(timedelta(hours=1))
        .with_grants(api.VideoGrants(room_join=True, room=body.room))
        .to_jwt()
    )
    return TokenResp(access_token=jwt, server_url=settings.livekit_url)
```

## Agent dispatch via token

Embed `RoomConfiguration.agents` in the token to auto-dispatch a named agent on join — frontend only needs the JWT.

## Identity rules

- `identity` is unique per room; reusing kicks the prior session.
- Treat as opaque — encode app user IDs but never raw PII.

## Tags
[[LiveKit]] [[Reference]] [[Auth]] [[JWT]] [[Tokens]]
