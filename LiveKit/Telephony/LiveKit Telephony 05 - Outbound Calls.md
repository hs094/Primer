# 05 — Outbound Calls

🔑 `CreateSIPParticipant` against an outbound trunk dials a number and drops the callee into a room as a participant. `wait_until_answered` blocks until 200 OK or raises `TwirpError` on rejection/timeout.

Source: https://docs.livekit.io/sip/outbound-calls/

## Prerequisites
- One configured `SIPOutboundTrunk` (see note 03).
- A target room (created on demand if it doesn't exist).

## Core API

```python
from livekit import api

lkapi = api.LiveKitAPI()

participant = await lkapi.sip.create_sip_participant(
    api.CreateSIPParticipantRequest(
        sip_trunk_id="ST_xxx",
        sip_call_to="+15551234567",
        room_name="outbound-call-42",
        participant_identity="caller-+15551234567",
        participant_name="Bob",
        wait_until_answered=True,
        play_dialtone=False,
    )
)
```

## Key parameters
| Field | Purpose |
|---|---|
| `sip_trunk_id` | Outbound trunk ID. |
| `sip_call_to` | E.164 destination. |
| `sip_number` | Caller-ID to use (only when trunk `numbers=["*"]`). |
| `room_name` | Target room. |
| `participant_identity` | Unique identity inside the room. |
| `participant_name` | Display name. |
| `wait_until_answered` | Block until 200 OK; otherwise raise `TwirpError`. |
| `play_dialtone` | Play dialtone to the room while ringing. |
| `dtmf` | DTMF to send after answer, e.g. `"*123#ww456"` (`w` = 0.5s wait). |
| `krisp_enabled` | Noise-cancel inbound audio from callee. |
| `headers` | Custom SIP headers (e.g. `X-...`). |

## Outcomes when `wait_until_answered=True`
- **200 OK** — returns `SIPParticipantInfo`.
- **486 Busy / 603 Decline** — raises `TwirpError` with SIP status.
- **408 Timeout / 480 Unavailable** — raises `TwirpError`.
- **Voicemail** — answers with 200 OK at SIP layer; you won't get a `TwirpError`. Detect via STT/IVR logic.

## Agent-driven outbound (dispatch pattern)

```python
# 1) Dispatch the agent with metadata containing the phone number
from livekit import api

dispatch = await lkapi.agent_dispatch.create_dispatch(
    api.CreateAgentDispatchRequest(
        agent_name="outbound-caller",
        room="outbound-call-42",
        metadata='{"phone_number":"+15551234567"}',
    )
)

# 2) Inside the agent entrypoint, parse metadata and place the call
async def entrypoint(ctx):
    meta = json.loads(ctx.job.metadata)
    await ctx.api.sip.create_sip_participant(
        api.CreateSIPParticipantRequest(
            sip_trunk_id="ST_xxx",
            sip_call_to=meta["phone_number"],
            room_name=ctx.room.name,
            participant_identity="callee",
            wait_until_answered=True,
        )
    )
```

## Caller ID
Set `display_name` on the request, or rely on the trunk's `numbers[0]`. Provider must permit the chosen CLI.

## Tags
[[LiveKit]] [[Telephony]] [[SIP]] [[Outbound]] [[Agents]]
