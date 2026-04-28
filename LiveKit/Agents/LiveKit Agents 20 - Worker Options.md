# 20 — Worker Options

🔑 `WorkerOptions` is the single configuration object that controls dispatch, permissions, prewarm, load-balancing, and shutdown behavior.

Source: https://docs.livekit.io/agents/worker/options/

## Core Fields
| Field | Purpose |
| --- | --- |
| `entrypoint_fnc` | `async def(ctx: JobContext)` — main per-job handler |
| `request_fnc` | Accept/reject a `JobRequest` before entrypoint runs |
| `prewarm_fnc` | Warm a `JobProcess` (load models) before assignment |
| `load_fnc` | Returns 0.0–1.0 load; default = 5s CPU avg |
| `load_threshold` | Stop accepting jobs above this load (default 0.7) |
| `job_memory_warn_mb` / `job_memory_limit_mb` | Per-job RSS limits |
| `num_idle_processes` | Pool of pre-forked, prewarmed workers |
| `shutdown_process_timeout` | Default 60s |
| `drain_timeout` | Grace period on SIGTERM (default ~30 min) |
| `agent_name` | Required for explicit dispatch |
| `worker_type` | `WorkerType.ROOM` or `WorkerType.PUBLISHER` |
| `permissions` | `WorkerPermissions(...)` |
| `host` / `port` | Health check HTTP server (prod default 8081) |

## Example
```python
from livekit.agents import WorkerOptions, WorkerPermissions, JobProcess, cli

def prewarm(proc: JobProcess) -> None:
    proc.userdata["vad"] = silero.VAD.load()

opts = WorkerOptions(
    entrypoint_fnc=entrypoint,
    prewarm_fnc=prewarm,
    agent_name="support-bot",
    load_threshold=0.75,
    num_idle_processes=2,
    permissions=WorkerPermissions(
        can_publish=True,
        can_subscribe=True,
        can_publish_data=True,
        can_update_metadata=True,
        hidden=False,
    ),
)
cli.run_app(opts)
```

## CLI Flags
- `--log-level` — `DEBUG | INFO | WARNING | ERROR | CRITICAL`
- `--url`, `--api-key`, `--api-secret` — override env
- `dev` runs single-process with hot reload; `start` runs production multi-process.

## Tags
[[LiveKit]] [[Agents]] [[WorkerOptions]] [[Configuration]] [[Deployment]]
