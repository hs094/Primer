# 01 — Introduction

🔑 LiveKit Agents is a realtime Python/Node.js framework that runs your code as a full participant inside a LiveKit room, bridging users and AI models over WebRTC.

Source: https://docs.livekit.io/agents/

## What it is
Open-source (Apache 2.0) framework for building voice, video, and physical AI agents. An agent is a stateful program that joins a LiveKit room as a participant and streams audio, video, and data to and from users in realtime.

## Core responsibilities the framework handles
- Streaming audio through STT-LLM-TTS pipelines
- Turn detection and interruption handling
- LLM orchestration (tool calls, chat context, handoffs)
- Agent server orchestration, load balancing, Kubernetes-friendly deployment

## Two model architectures
- **STT-LLM-TTS pipeline** — discrete speech-to-text, LLM, text-to-speech components (more control, swappable parts)
- **Realtime model** — single multimodal model (e.g. OpenAI Realtime API) for lower latency and more natural prosody

## Minimal shape of an agent
```python
from livekit.agents import Agent, AgentSession, JobContext, WorkerOptions, cli

class Assistant(Agent):
    def __init__(self) -> None:
        super().__init__(instructions="You are a helpful voice assistant.")

async def entrypoint(ctx: JobContext) -> None:
    session = AgentSession(...)
    await session.start(agent=Assistant(), room=ctx.room)

if __name__ == "__main__":
    cli.run_app(WorkerOptions(entrypoint_fnc=entrypoint))
```

## Deployment
LiveKit Cloud managed infrastructure, or self-host. Common workloads: multimodal assistants, telehealth, call centers, realtime translation, NPCs, robotics.

## Tags
[[LiveKit]] [[Agents]] [[Realtime]] [[WebRTC]] [[VoiceAI]]
