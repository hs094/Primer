# 07 — DTMF and IVR

🔑 DTMF rides RTP `telephone-event/8000` (RFC 2833) — send via `publish_dtmf`, receive via room events. Agents framework has `ivr_detection` + `GetDtmfTask` for menu navigation.

Source: https://docs.livekit.io/sip/dtmf/

## Sending DTMF (out of agent → callee)

```python
# LocalParticipant.publish_dtmf(code, digit)
await ctx.room.local_participant.publish_dtmf(code=1, digit="1")
await ctx.room.local_participant.publish_dtmf(code=11, digit="#")
```

`code` is the numeric event code; `digit` is the string form (`"0"`–`"9"`, `"*"`, `"#"`, `"A"`–`"D"`).

You can also pre-load DTMF when placing an outbound call:

```python
api.CreateSIPParticipantRequest(
    sip_trunk_id="ST_xxx",
    sip_call_to="+15551234567",
    room_name="r",
    participant_identity="callee",
    dtmf="*123#ww456",   # 'w' = 0.5s pause
)
```

## Receiving DTMF (callee → agent)
SIP relays DTMF as room events on the participant.

```python
from livekit import rtc

@ctx.room.on("sip_dtmf_received")
def on_dtmf(participant: rtc.RemoteParticipant, code: int, digit: str) -> None:
    print(f"got {digit} from {participant.identity}")
```

## Agents framework helpers

### `ivr_detection`
AgentSession option that auto-detects IVR prompts and forwards model-decided DTMF back to the carrier.

```python
session = AgentSession(
    ...,
    ivr_detection=True,
)
```

### `GetDtmfTask`
Prebuilt task that prompts the caller and collects DTMF *or* spoken digits until a terminator/timeout.

```python
from livekit.agents.voice.tasks import GetDtmfTask

digits = await GetDtmfTask(
    instructions="Enter your 4-digit PIN followed by #.",
    terminator="#",
    max_digits=4,
).run(session)
```

## Wire format
- Payload type: `telephone-event/8000` (RFC 2833 / RFC 4733).
- Codec-independent: tones are signaled, not synthesized into PCM.
- Reliable across Opus, G.711, etc.

## Tags
[[LiveKit]] [[Telephony]] [[SIP]] [[DTMF]] [[IVR]] [[Agents]]
