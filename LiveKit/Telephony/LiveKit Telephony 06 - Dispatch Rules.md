# 06 — Dispatch Rules

🔑 Dispatch rules decide which room an inbound caller lands in, what attributes they carry, and which agents auto-dispatch. Three flavors: Individual, Direct, Callee.

Source: https://docs.livekit.io/sip/dispatch-rule/

## Rule types

### Individual — one room per caller
Generates a fresh room name per call (`<prefix>_<phone>_<random>`). Best for 1:1 agent calls.

```json
{
  "name": "per-caller",
  "rule": {
    "dispatchRuleIndividual": {
      "roomPrefix": "call-"
    }
  },
  "trunk_ids": [],
  "hide_phone_number": false,
  "room_config": {
    "agents": [
      {"agent_name": "receptionist", "metadata": "{}"}
    ]
  }
}
```

### Direct — all callers into one named room
For shared lobbies / conferences. Optional PIN.

```json
{
  "rule": {
    "dispatchRuleDirect": {
      "roomName": "support-lobby",
      "pin": "1234"
    }
  }
}
```

### Callee — room per called number
Routes by **DID** (the number that was dialed), with optional random suffix per caller.

```json
{
  "rule": {
    "dispatchRuleCallee": {
      "roomPrefix": "callee-",
      "randomize": true
    }
  }
}
```

## Common fields
- `trunk_ids` — restrict rule to specific inbound trunks; empty = all trunks.
- `hide_phone_number` — strip caller's number from participant attributes shown to other participants.
- `attributes` — custom k/v added to the SIP participant.
- `metadata` — string metadata on the participant.
- `room_config.agents[]` — `RoomAgentDispatch` entries auto-dispatched into the room.
- `room_config.empty_timeout`, `max_participants`, etc. — pass-through to room creation.

## Python create

```python
from livekit import api

await lkapi.sip.create_sip_dispatch_rule(
    api.CreateSIPDispatchRuleRequest(
        dispatch_rule=api.SIPDispatchRuleInfo(
            name="per-caller",
            rule=api.SIPDispatchRule(
                dispatch_rule_individual=api.SIPDispatchRuleIndividual(
                    room_prefix="call-",
                ),
            ),
            room_config=api.RoomConfiguration(
                agents=[api.RoomAgentDispatch(agent_name="receptionist")],
            ),
        )
    )
)
```

## Management
- `ListSIPDispatchRule`
- `UpdateSIPDispatchRule` — replace
- `DeleteSIPDispatchRule`

## Tags
[[LiveKit]] [[Telephony]] [[SIP]] [[DispatchRules]] [[Agents]]
