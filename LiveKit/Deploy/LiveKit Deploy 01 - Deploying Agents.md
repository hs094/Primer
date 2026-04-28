# 01 — Deploying Agents

🔑 `lk agent` ships a Dockerfile-based build to LiveKit Cloud's global edge with autoscaling, encrypted secrets, and log drains; bring your own infra by running the same worker as a long-lived process.

Source: https://docs.livekit.io/agents/ops/deployment/ (also reachable at https://docs.livekit.io/agents/deployment/)

## LiveKit Cloud Deploy (recommended)

```bash
# one-time
lk cloud auth

# from the agent project root (must contain Dockerfile)
lk agent create
lk agent deploy
lk agent status
lk agent rollback <version>
lk agent delete
```

Each `deploy` builds the Docker image, uploads it, and rolls workers behind a health check.

## Dockerfile pattern

```dockerfile
FROM python:3.12-slim
WORKDIR /app

RUN pip install --no-cache-dir uv
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev

COPY . .

# Required: download/cache models so cold-start is fast
RUN uv run python agent.py download-files

CMD ["uv", "run", "python", "agent.py", "start"]
```

## Worker Entrypoint

```python
import logging
from livekit.agents import WorkerOptions, cli, JobContext

logger = logging.getLogger(__name__)

async def entrypoint(ctx: JobContext) -> None:
    await ctx.connect()
    # ... build Agent + AgentSession here
    logger.info(f"agent connected to room {ctx.room.name}")

if __name__ == "__main__":
    cli.run_app(WorkerOptions(entrypoint_fnc=entrypoint))
```

CLI subcommands: `start`, `dev`, `connect --room <name>`, `download-files`.

## Secrets

```bash
lk agent secrets set OPENAI_API_KEY=sk-...
lk agent secrets list
lk agent secrets delete OPENAI_API_KEY
```

Injected as env vars at runtime; encrypted at rest.

## Log Drains

Forward runtime + session logs to Datadog, CloudWatch, Sentry, or New Relic from the dashboard. Three log streams: `build`, `runtime`, `session`.

## Dispatch

Default: agent auto-dispatches on every room. Explicit dispatch (per-room control):

```python
from livekit import api

req = api.CreateAgentDispatchRequest(
    agent_name="support-bot",
    room="room-123",
    metadata='{"customer_id": "abc"}',
)
await api.LiveKitAPI().agent_dispatch.create_dispatch(req)
```

Set `agent_name=` in `WorkerOptions` to opt out of auto-dispatch.

## Self-Hosted Agents

Run the worker container against any LiveKit URL (Cloud or self-hosted server). Health check: `GET :8081/` returns 200 when the worker can accept jobs.

## Tags
[[LiveKit]] [[Deploy]] [[Agents]]
