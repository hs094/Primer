# 09 — Authentication and Tokens

🔑 LiveKit auth = JWT signed with API key/secret, carrying a `VideoGrant` and identity. Mint server-side, deliver to client, hand to `Room.connect`.

Source: https://docs.livekit.io/home/get-started/authentication/

## Token Anatomy
- `iss` — API key
- `sub` — participant identity
- `exp` — expiry (short-lived; 10 min – 6 h typical)
- `name`, `metadata`, `attributes` — surface on the participant
- `video` — `VideoGrant` (room name + permissions)
- `roomConfig` — agent dispatch, codecs, recording

## VideoGrant Fields
- `room: string` — room name
- `roomJoin: bool`
- `roomCreate`, `roomList`, `roomAdmin`, `roomRecord`
- `canPublish`, `canSubscribe`, `canPublishData`
- `canPublishSources: TrackSource[]` — restrict to e.g. `[Microphone]`
- `canUpdateOwnMetadata`
- `hidden` — invisible participant
- `recorder` — egress process

## Python (server-sdk)
```python
from datetime import timedelta
from livekit import api

token = (
    api.AccessToken(api_key, api_secret)
    .with_identity("user-42")
    .with_name("Alice")
    .with_attributes({"role": "host"})
    .with_ttl(timedelta(hours=1))
    .with_grants(
        api.VideoGrants(
            room_join=True,
            room="lobby",
            can_publish=True,
            can_subscribe=True,
            can_publish_data=True,
        )
    )
    .to_jwt()
)
```

## Node.js (server-sdk-js)
```typescript
import { AccessToken } from 'livekit-server-sdk';

const at = new AccessToken(apiKey, apiSecret, {
  identity: 'user-42',
  name: 'Alice',
  ttl: '1h',
});
at.addGrant({
  room: 'lobby',
  roomJoin: true,
  canPublish: true,
  canSubscribe: true,
});
const jwt = await at.toJwt();
```

## TokenSource Variants (Frontend)
- `TokenSource.literal({ serverUrl, participantToken })` — pre-issued JWT.
- `TokenSource.endpoint({ url, headers? })` — backend endpoint returns `{ serverUrl, participantToken }`; SDK refreshes automatically.
- `TokenSource.custom(async () => …)` — plug into existing auth (Auth0, Clerk, Supabase).
- `TokenSource.sandbox({ sandboxId })` — Cloud-issued tokens, dev only.

## FastAPI Token Endpoint
```python
from fastapi import APIRouter, Depends
from livekit import api
from .deps import get_user, settings

router = APIRouter()

@router.post("/livekit/token")
async def issue_token(room: str, user=Depends(get_user)) -> dict[str, str]:
    token = (
        api.AccessToken(settings.livekit_api_key, settings.livekit_api_secret)
        .with_identity(user.id)
        .with_grants(api.VideoGrants(room_join=True, room=room))
        .to_jwt()
    )
    return {"serverUrl": settings.livekit_url, "participantToken": token}
```

## Auth Flow
1. Backend mints JWT after authenticating the user.
2. Frontend receives `{ serverUrl, participantToken }`.
3. SDK passes both to `Room.connect` (or `Session`).
4. Server validates signature + grants, opens WebSocket, dispatches agents per `roomConfig`.

## Tags
[[LiveKit]] [[JWT]] [[Auth]] [[VideoGrant]]
