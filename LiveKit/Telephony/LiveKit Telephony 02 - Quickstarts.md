# 02 — Quickstarts

🔑 Pick a path: LiveKit Phone Numbers (one-step) or BYO SIP provider (trunks + dispatch rule + agent).

Source: https://docs.livekit.io/sip/quickstart/

## Path A — LiveKit Phone Numbers (Cloud only, US)
1. Buy a number in the LiveKit Cloud dashboard.
2. Create a dispatch rule pointing at your agent.
3. Done. No trunk setup required.

## Path B — Bring your own SIP provider
Compatible with Twilio, Telnyx, Plivo, Signalwire, etc.

1. Purchase a DID from the provider.
2. Configure the provider's SIP trunk (point termination URI at LiveKit's SIP gateway).
3. Create an inbound and/or outbound trunk in LiveKit.
4. Create a dispatch rule for inbound.
5. Build an agent that picks up the room.

## Minimal Python agent

```python
from livekit import agents
from livekit.agents import AgentSession, Agent
from livekit.plugins import openai, deepgram, silero

class Receptionist(Agent):
    def __init__(self) -> None:
        super().__init__(instructions="You are a phone receptionist.")

async def entrypoint(ctx: agents.JobContext) -> None:
    session = AgentSession(
        stt=deepgram.STT(),
        llm=openai.LLM(model="gpt-4o-mini"),
        tts=openai.TTS(),
        vad=silero.VAD.load(),
    )
    await session.start(agent=Receptionist(), room=ctx.room)

if __name__ == "__main__":
    agents.cli.run_app(agents.WorkerOptions(entrypoint_fnc=entrypoint))
```

## CLI helpers
```bash
lk sip inbound create inbound-trunk.json
lk sip outbound create outbound-trunk.json
lk sip dispatch create dispatch-rule.json
lk sip participant create participant.json   # outbound call
```

## Tags
[[LiveKit]] [[Telephony]] [[SIP]] [[Quickstart]] [[Agents]]
