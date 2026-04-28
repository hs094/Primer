# 22 — Agent Dispatch

🔑 Set `agent_name` for explicit dispatch in production; automatic dispatch is convenient but joins every room.

Source: https://docs.livekit.io/agents/worker/agent-dispatch/

## Three Methods

### 1. Explicit Dispatch (recommended)
Set `agent_name` in `WorkerOptions`, then call the dispatch API per room. Lets you pass `metadata` (user id, phone number, scenario flags).

```python
from livekit import api

lkapi = api.LiveKitAPI()
await lkapi.agent_dispatch.create_dispatch(
    api.CreateAgentDispatchRequest(
        agent_name="support-bot",
        room="room-abc",
        metadata='{"user_id": "u_123"}',
    )
)
```

### 2. Automatic Dispatch
Omit `agent_name`. The worker is assigned to every newly created room. Easy for prototypes; not recommended for production (deploys agents you may not need).

### 3. Token-Based Dispatch
Embed dispatch entries in a participant access token. Only fires when the room is first created.

```python
token = (api.AccessToken()
    .with_identity("user-1")
    .with_room_config(api.RoomConfiguration(
        agents=[api.RoomAgentDispatch(agent_name="support-bot", metadata="...")]
    ))
    .to_jwt())
```

## Reading Metadata
```python
async def entrypoint(ctx: JobContext) -> None:
    payload = json.loads(ctx.job.metadata or "{}")
    user_id = payload.get("user_id")
```

## Performance
Dispatch handles hundreds of thousands of new connections per second with max dispatch latency under ~150 ms.

## SIP Integration
For inbound SIP, attach `room_config.agents` in the dispatch rule so the agent joins the moment the call lands.

## Tags
[[LiveKit]] [[Agents]] [[Dispatch]] [[AgentDispatch]] [[SIP]]
