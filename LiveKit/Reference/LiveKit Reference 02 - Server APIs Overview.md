# 02 — Server APIs Overview

🔑 Server SDKs (Go, Node, Python, Ruby) wrap the LiveKit gRPC/HTTP control plane — `RoomServiceClient` is the entry point for room/participant/egress/ingress operations.

Source: https://docs.livekit.io/home/get-started/intro-to-livekit/ (canonical /home/server/ returns 404)

## SDK matrix

- **Server SDKs**: Go, Node.js, Python, Ruby — for backend control-plane calls (create rooms, mint tokens, manage participants, start egress/ingress).
- **Client SDKs**: JavaScript, Swift, Android, Flutter, React Native — for end-user media.
- **Agent frameworks**: `livekit-agents` (Python), `@livekit/agents` (Node).

## Python client init

```python
from livekit import api

lkapi = api.LiveKitAPI(
    url="https://my-project.livekit.cloud",
    api_key="API_KEY",
    api_secret="API_SECRET",
)

# Sub-services exposed on the client
lkapi.room          # RoomService
lkapi.egress        # EgressService
lkapi.ingress       # IngressService
lkapi.sip           # SIPService
lkapi.agent_dispatch
```

## Auth model

Every server call is signed with an `AccessToken` containing a `VideoGrants` payload (`roomCreate`, `roomList`, `roomAdmin`, `roomRecord`, etc.). Server SDKs sign automatically from `api_key` / `api_secret`.

## Use cases

- Mint room-join JWTs for clients.
- Create/list/delete rooms.
- Mutate participant permissions, metadata, tracks.
- Trigger egress recordings and ingress streams.
- Receive webhooks for lifecycle events.

## Tags
[[LiveKit]] [[Reference]] [[ServerSDK]] [[API]]
