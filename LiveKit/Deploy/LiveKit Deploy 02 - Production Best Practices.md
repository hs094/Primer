# 02 — Production Best Practices

🔑 Tune `WorkerOptions` (load function, idle pool, drain timeout), prewarm heavy resources, and load-test with `lk perf agent-load-test` to find the inflection point before users do.

Source: https://docs.livekit.io/agents/ops/production/ (404 at fetch time — content stitched from https://docs.livekit.io/agents/worker/options/ and https://docs.livekit.io/agents/start/testing/)

## WorkerOptions

```python
from livekit.agents import WorkerOptions, JobContext, JobProcess

def prewarm(proc: JobProcess) -> None:
    # loaded once per worker process, reused across jobs
    proc.userdata["vad"] = silero.VAD.load()

async def entrypoint(ctx: JobContext) -> None:
    vad = ctx.proc.userdata["vad"]
    ...

opts = WorkerOptions(
    entrypoint_fnc=entrypoint,
    prewarm_fnc=prewarm,
    agent_name="support",          # disables auto-dispatch
    num_idle_processes=3,           # warm pool
    load_threshold=0.7,             # stop accepting jobs above this
    load_fnc=None,                  # default: CPU util in [0,1]
    drain_timeout=1800,             # 30m graceful shutdown
    job_memory_warn_mb=500,
    job_memory_limit_mb=0,          # 0 = no limit
    permissions=WorkerPermissions(can_publish=True, can_subscribe=True, hidden=False),
    worker_type=WorkerType.ROOM,    # or WorkerType.PUBLISHER
)
```

`WorkerType.ROOM` — one job per room (default). `WorkerType.PUBLISHER` — one job per publishing participant (e.g., per-camera processing).

## Prewarm vs Cold Start

Move expensive imports + model loads into `prewarm_fnc`:
- VAD models (`silero.VAD.load()`)
- Tokenizers
- Vector DB clients

The pool keeps `num_idle_processes` warm; new jobs grab a ready process and skip cold start entirely.

## Graceful Shutdown

LiveKit Cloud and `lk agent deploy` send SIGTERM; the worker stops accepting new jobs and waits up to `drain_timeout` for in-flight sessions to finish. In Kubernetes, set `terminationGracePeriodSeconds` >= `drain_timeout`.

## Load Testing

```bash
lk perf agent-load-test \
  --url wss://your.livekit.cloud \
  --api-key ... --api-secret ... \
  --rooms 10 \
  --duration 5m
```

Process: start at 10 rooms, double until response latency or join time degrades. Test at 2x peak. Use synthetic callers with realistic noise for sessions > 100 concurrent.

## Health Check

Each worker exposes `GET :8081/` (configurable). Returns 200 if `current_load < load_threshold`, else 503. Wire to Kubernetes `readinessProbe`.

## Logging

```python
import logging
logger = logging.getLogger(__name__)
logger.info(f"job assigned room={ctx.room.name} job_id={ctx.job.id}")
```

Use stdlib `logging`, JSON formatter in prod, ship via log drain.

## Resource Sizing

- 1 vCPU per ~5-10 concurrent voice sessions (depends on TTS/STT provider).
- Memory: budget 200-500 MB per active job.
- Network: voice is ~50 kbps per leg; trivial vs CPU.

## Tags
[[LiveKit]] [[Deploy]] [[Production]] [[Agents]]
