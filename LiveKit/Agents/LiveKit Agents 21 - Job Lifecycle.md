# 21 — Job Lifecycle

🔑 Each accepted job runs in its own subprocess; the entrypoint owns the room until all participants leave or shutdown is invoked.

Source: https://docs.livekit.io/agents/worker/job/

## Stages
1. **Request** — `request_fnc(req: JobRequest)` decides `await req.accept()` or `await req.reject()`.
2. **Process pickup** — A prewarmed `JobProcess` is assigned (or a new one forked).
3. **Entrypoint** — `entrypoint_fnc(ctx: JobContext)` executes as the process main.
4. **Connect** — `await ctx.connect()` joins the room.
5. **Run** — Session loops until participants leave or you call shutdown.
6. **Shutdown** — Registered callbacks run inside `shutdown_process_timeout`.

## JobContext Surface
```python
async def entrypoint(ctx: JobContext) -> None:
    await ctx.connect()
    participant = await ctx.wait_for_participant()
    metadata: str = ctx.job.metadata          # dispatch payload
    room = ctx.room                            # rtc.Room
    ctx.add_shutdown_callback(flush_metrics)   # async cleanup
    ctx.add_participant_entrypoint(on_join)    # per-participant hooks
```

Useful attrs: `ctx.room`, `ctx.job`, `ctx.proc.userdata` (shared with prewarm).

## Prewarm Pattern
```python
def prewarm(proc: JobProcess) -> None:
    proc.userdata["vad"] = silero.VAD.load()

async def entrypoint(ctx: JobContext) -> None:
    vad = ctx.proc.userdata["vad"]
    session = AgentSession(vad=vad, ...)
```

## Shutdown
- `await session.aclose()` — immediate close.
- `session.shutdown(drain=True)` — non-blocking; finishes pending speech first.
- Shutdown callbacks must complete in ~10s default; raise `drain_timeout` for longer flushes.

## Data Inputs
Pass per-job context via:
- `ctx.job.metadata` (string set on dispatch)
- `ctx.room.metadata` / `ctx.room.name`
- `participant.attributes` / `participant.metadata`

## Tags
[[LiveKit]] [[Agents]] [[JobContext]] [[Lifecycle]] [[Prewarm]]
