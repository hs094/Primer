# 04 — Inbound Calls

🔑 Provider sends INVITE to LiveKit SIP, which matches an inbound trunk, applies a dispatch rule to pick a room, then creates a SIP participant — your agent treats it like any other participant.

Source: https://docs.livekit.io/sip/accepting-calls/

## Flow
```
Caller ──▶ Provider ──INVITE──▶ LiveKit SIP
                                    │
                                    ├── match SIPInboundTrunk by DID
                                    ├── apply SIPDispatchRule → room name + attrs
                                    └── create SIP Participant in Room
                                                    │
                                                    └── Agent worker dispatched
```

## Required pieces
1. A configured **inbound trunk** (`SIPInboundTrunkInfo`).
2. A **dispatch rule** (`SIPDispatchRule`) that targets that trunk (or all).
3. An **agent** (or any participant) in the destination room.

## Participant attributes set on inbound SIP participants
- `sip.callID` — provider Call-ID.
- `sip.trunkID` — matched inbound trunk.
- `sip.phoneNumber` — caller's number (E.164).
- `sip.trunkPhoneNumber` — the called DID.
- `sip.ruleID` — matched dispatch rule.
- Plus any custom attributes from the dispatch rule.

## Minimal agent that answers

```python
from livekit import agents
from livekit.agents import Agent, AgentSession, JobContext

class Receptionist(Agent):
    def __init__(self) -> None:
        super().__init__(instructions="Greet the caller and help them.")

async def entrypoint(ctx: JobContext) -> None:
    await ctx.connect()
    # Inbound SIP participant is already (or about to be) in the room.
    session = AgentSession(...)
    await session.start(agent=Receptionist(), room=ctx.room)

    # Optional: read SIP attributes
    for p in ctx.room.remote_participants.values():
        caller = p.attributes.get("sip.phoneNumber")
        ctx.log.info(f"caller is {caller}")
```

## Twilio specifics
TwiML can hand a call into a LiveKit room via SIP REFER or by configuring the Twilio trunk's termination URI to point at LiveKit's SIP ingress. The doc covers using `<Dial><Sip>` and conference bridging when you need to merge legs.

## Simplification
With **LiveKit Phone Numbers** (Cloud), the trunk is implicit — buy a number, attach a dispatch rule, done.

## Tags
[[LiveKit]] [[Telephony]] [[SIP]] [[Inbound]] [[Agents]]
