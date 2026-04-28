# 01 — Frontends Overview

🔑 LiveKit ships open-source MIT-licensed starter apps across six platforms, all driven by the same token-server + room-connect contract.

Source: https://docs.livekit.io/agents/start/frontend/

## Available Starter Apps

| Platform | Stack | Use case |
|----------|-------|----------|
| Next.js | React + Next.js App Router | Web voice AI assistant |
| SwiftUI | Native iOS / macOS / visionOS | Apple ecosystem |
| Android | Kotlin + Jetpack Compose | Native Android |
| Flutter | Dart | Cross-platform mobile |
| React Native | Expo | Mobile JS |
| Web Embed | Vanilla JS widget | Drop-in for any site |

## Connection Contract

Every frontend follows the same flow:

1. Frontend asks your token server for a JWT scoped to a room + identity.
2. Frontend calls `room.connect(wsUrl, token)`.
3. Agent worker is dispatched (auto on room create, or via `AgentDispatchService`).

Minimal token server (Python, FastAPI):

```python
from livekit import api
from fastapi import FastAPI
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    livekit_api_key: str
    livekit_api_secret: str
    livekit_url: str

settings = Settings()
app = FastAPI()

@app.get("/token")
async def token(room: str, identity: str) -> dict[str, str]:
    grant = api.VideoGrants(room_join=True, room=room)
    jwt = (
        api.AccessToken(settings.livekit_api_key, settings.livekit_api_secret)
        .with_identity(identity)
        .with_grants(grant)
        .to_jwt()
    )
    return {"token": jwt, "url": settings.livekit_url}
```

## Sandbox Token Server

For dev only, LiveKit Cloud Sandbox issues short-lived tokens without a backend — use it to prototype mobile apps before standing up your own service.

## Pre-built UI Pieces

Every starter ships:
- Mic / camera toggles bound to `LocalParticipant` track publish state.
- Live transcription overlay sourced from `room.on("transcription_received")`.
- Agent state pill (`listening`, `thinking`, `speaking`) from agent attributes.
- Audio visualizer driven by track audio level.

## Tags
[[LiveKit]] [[Frontends]] [[Agents]]
