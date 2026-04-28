# 08 — Call Transfer (Cold)

🔑 `TransferSIPParticipant` sends a SIP REFER through the trunk; the caller leaves the LiveKit room and the provider stitches them to a new PSTN/SIP destination. Agent's session ends.

Source: https://docs.livekit.io/sip/transfer-cold/

## Cold vs warm
- **Cold transfer**: caller is forwarded and the LiveKit session for that caller closes immediately. No introduction by the agent.
- **Warm transfer**: agent talks to the new party first, then bridges (covered separately).

## How it works
1. Agent calls `TransferSIPParticipant` with a target URI.
2. LiveKit sends `SIP REFER` to your provider on the existing dialog.
3. Provider sets up the new leg and tears down the LiveKit leg.
4. Caller's `RemoteParticipant` disconnects from the room.

## Python

```python
from livekit import api

lkapi = api.LiveKitAPI()

await lkapi.sip.transfer_sip_participant(
    api.TransferSIPParticipantRequest(
        room_name="call-42",
        participant_identity="caller-+15551234567",
        transfer_to="tel:+15559876543",          # or "sip:agent@host"
        play_dialtone=False,
    )
)
```

Parameters:
- `room_name`, `participant_identity` — identify the SIP participant.
- `transfer_to` — `tel:E164` or `sip:user@host` URI.
- `play_dialtone` — play dialtone to the caller during REFER.
- `headers` — extra SIP headers on the REFER.

## As an agent tool

```python
from livekit.agents import function_tool, RunContext

@function_tool
async def transfer_to_human(ctx: RunContext, number: str) -> str:
    """Cold-transfer the caller to a human."""
    await ctx.session.api.sip.transfer_sip_participant(
        api.TransferSIPParticipantRequest(
            room_name=ctx.room.name,
            participant_identity=ctx.caller_identity,
            transfer_to=f"tel:{number}",
        )
    )
    return "transferred"
```

## Provider notes
- **Twilio**: enable "Call Transfer" on the SIP trunk; choose whether REFER uses the original caller's CLI or the trunk's number.
- **Telnyx / others**: ensure REFER is permitted on the trunk profile.

## Tags
[[LiveKit]] [[Telephony]] [[SIP]] [[Transfer]] [[Agents]]
