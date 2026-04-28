# 10 — Sessions

🔑 `AgentSession` is the per-job orchestrator: it owns the STT/LLM/TTS pipeline, runs an active `Agent`, and exposes lifecycle methods for graceful shutdown.

Source: https://docs.livekit.io/agents/build/session/

## Job lifecycle
Each dispatched job runs in its own subprocess. The entrypoint receives a `JobContext`; you connect to the room, build an `AgentSession`, and `await session.start(...)`.

```python
from livekit.agents import AgentSession, JobContext, WorkerOptions, cli

async def entrypoint(ctx: JobContext) -> None:
    await ctx.connect()  # explicit connect — useful for E2EE timing
    session = AgentSession(
        stt=stt_provider,
        llm=llm_provider,
        tts=tts_provider,
        vad=vad_provider,
    )
    await session.start(room=ctx.room, agent=MyAgent())

    async def on_shutdown() -> None:
        await save_transcript(session.history)
    ctx.add_shutdown_callback(on_shutdown)  # 10s default timeout

cli.run_app(WorkerOptions(entrypoint_fnc=entrypoint))
```

## Shutdown
- `session.shutdown(drain=True)` — non-blocking; drains pending speech.
- `await session.aclose()` — immediate async close; awaits completion.
- `delete_room_on_close=True` on session config auto-deletes the room.

## Passing data into a job
- **Job metadata**: freeform string (typically JSON) on dispatch.
- **Room metadata**: `ctx.room.name`, `ctx.room.metadata`.
- **Participant attributes**: `participant.attributes.get("user_id")`.

Prefer prewarm functions for static data; pass user-specific data via job metadata rather than fetching in `entrypoint`.

## Tags
[[LiveKit]] [[Agents]] [[AgentSession]] [[JobContext]] [[Lifecycle]]
