# 28 — Virtual Avatars

🔑 An `AvatarSession` joins the room as a secondary participant, consumes the agent's audio, and publishes a synced talking-head video track.

Source: https://docs.livekit.io/agents/integrations/avatar/

## Providers
14+ vendors: Anam, Tavus, Beyond Presence (Bey), Hedra, bitHuman, Simli, D-ID, Runway, TruGen, LemonSlice, and more. Most are Python-only; Anam, Beyond Presence, LemonSlice, Runway, TruGen, and Hedra also ship Node.

## Wiring Pattern
```python
from livekit.agents import AgentSession, RoomOutputOptions
from livekit.plugins import tavus

async def entrypoint(ctx: JobContext) -> None:
    await ctx.connect()

    session = AgentSession(stt=..., llm=..., tts=...)

    avatar = tavus.AvatarSession(
        replica_id="r_abc",
        persona_id="p_xyz",
    )
    await avatar.start(session, room=ctx.room)

    await session.start(
        agent=MyAgent(),
        room=ctx.room,
        # critical: avatar consumes the audio, room must not also publish it
        room_output_options=RoomOutputOptions(audio_enabled=False),
    )
```

The avatar provider receives the session's audio output, renders lip-synced video, and publishes both audio + video back into the room.

## Frontend Detection
Two agent participants now exist in the room. Distinguish them on the client:
- Main agent: `participant.kind === 'agent'`
- Avatar worker: also `kind === 'agent'`, plus the `lk.publish_on_behalf` attribute pointing at the main agent's identity.

React: `useVoiceAssistant()` already returns the correct audio + video tracks automatically.

## Tags
[[LiveKit]] [[Agents]] [[Avatar]] [[AvatarSession]] [[Tavus]] [[Hedra]]
