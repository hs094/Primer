# 04 — Connecting

🔑 Connect with a `wsUrl` + per-participant access token; LiveKit walks an ICE/TURN ladder for resilience.

Source: https://docs.livekit.io/intro/basics/connect/

## Required Parameters
| Param | Source |
|---|---|
| `wsUrl` | LiveKit Cloud Project Settings, or `ws://localhost:7880` self-hosted |
| `token` | JWT encoding room name, participant identity, permissions — generated server-side per participant |

## Token Generation (Python)
```python
from livekit import api

token = (
    api.AccessToken(api_key, api_secret)
    .with_identity("user-123")
    .with_name("Alice")
    .with_grants(api.VideoGrants(room_join=True, room="my-room"))
    .to_jwt()
)
```

## Room Object
After connect, the `Room` exposes:
- `localParticipant` — current user/agent
- `remoteParticipants` — map of other participants

## Network Fallback Order
1. ICE over UDP
2. TURN with UDP
3. ICE over TCP
4. TURN with TLS

On network change: WebSocket reconnect + ICE restart, usually minimal disruption.

## Disconnect Reasons
Server emits a `Disconnected` event with a reason:
- `DUPLICATE_IDENTITY`
- `ROOM_DELETED`
- `PARTICIPANT_REMOVED`
- (plus others — server-initiated or manual)

## SDK Coverage
- **Frontend**: JavaScript, Swift, Android, React Native, Flutter, Unity
- **Backend**: Python, Node.js, Go, Rust, Ruby, Kotlin

## Tags
[[LiveKit]] [[WebRTC]] [[Access Token]] [[ICE]] [[TURN]] [[Connection]]
