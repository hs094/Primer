# 02 — Connecting to LiveKit

🔑 Connect with `Room.connect(wsUrl, token)`; the Session API wraps token acquisition + lifecycle for production use.

Source: https://docs.livekit.io/home/client/connect/

## Required Inputs
- `wsUrl` — WebSocket URL (`wss://<project>.livekit.cloud` in cloud, `ws://localhost:7880` for self-hosted dev).
- `token` — JWT encoding room name, identity, and grants. Always minted on a backend.

## Direct `Room.connect` (JS/TS)
```typescript
import { Room, RoomEvent } from 'livekit-client';

const room = new Room();
await room.connect(wsUrl, token);

console.log(room.localParticipant.identity);
for (const p of room.remoteParticipants.values()) {
  console.log('peer:', p.identity);
}

await room.disconnect();
```

## Direct connect (Python, agents/realtime)
```python
from livekit import rtc

room = rtc.Room()
await room.connect(ws_url, token)
print(f"local: {room.local_participant.identity}")
await room.disconnect()
```

## Session API (recommended for frontends)
```typescript
import { TokenSource, useSession } from '@livekit/components-react';

const tokenSource = TokenSource.literal({
  serverUrl: wsUrl,
  participantToken: token,
});
const session = useSession(tokenSource);
```
`TokenSource` variants: `literal`, `endpoint`, `custom`, `sandbox`. Endpoint/custom enable automatic refresh and caching.

## Disconnect Semantics
Call `room.disconnect()` to leave cleanly. Without it, the server reaps the participant after a ~15s grace window once the WebSocket drops.

## Tags
[[LiveKit]] [[WebRTC]] [[JWT]] [[Session]]
