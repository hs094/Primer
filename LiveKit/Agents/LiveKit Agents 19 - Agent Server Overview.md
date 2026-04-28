# 19 — Agent Server Overview

🔑 The agent server is a long-running process that registers with LiveKit, accepts job requests, and spawns isolated subprocesses for each room session.

Source: https://docs.livekit.io/agents/worker/

## Architecture
The Agents framework is a server runtime, not just a library. A single server process:
- Registers with LiveKit (Cloud or self-hosted) over a persistent connection.
- Receives job offers when rooms need an agent.
- Forks an isolated subprocess per accepted job (crash isolation, parallel sessions).
- Reports load + drains gracefully on shutdown.

## Programmatic Participants
"Agent" is anything that joins a room programmatically — not only LLM voice bots. Use cases:
- Audio analysis (quality metrics, pattern detection).
- Computer vision / video effects.
- Realtime data transformation.
- Bridges to external systems.

You can run an entrypoint without ever creating an `AgentSession`; the worker still gets dispatch + scaling.

## Core Components
| Component | Purpose |
| --- | --- |
| Agent Dispatch | Assigns agents to rooms with load balancing |
| Job Lifecycle | Runs entrypoint, manages cleanup |
| Worker Options | Permissions, dispatch rules, prewarm, load fn |

## Minimal Entrypoint
```python
from livekit.agents import JobContext, WorkerOptions, cli

async def entrypoint(ctx: JobContext) -> None:
    await ctx.connect()
    # session logic here

if __name__ == "__main__":
    cli.run_app(WorkerOptions(entrypoint_fnc=entrypoint))
```

`cli.run_app` provides `dev`, `start`, `connect`, and `download-files` subcommands.

## Tags
[[LiveKit]] [[Agents]] [[Worker]] [[Architecture]] [[Server]]
