# 06 — Building AI Agents

🔑 LiveKit agents are room participants written in Python or Node.js that process realtime audio/video/data and respond multimodally.

Source: https://docs.livekit.io/intro/basics/agents/

## Mental Model
Agents join a `Room` exactly like a human participant. They subscribe to other participants' tracks, run inference (STT → LLM → TTS, or end-to-end multimodal), and publish their own audio/video/data tracks back.

## Capabilities
- Realtime audio, video, and data stream processing
- Python or Node.js runtimes
- Pluggable AI providers (LLM, STT, TTS, vision)
- Multimodal input: speech, text, vision

## Two Starting Paths
| Path | Best for |
|---|---|
| **Voice AI Quickstart** | Working voice assistant in under 10 minutes |
| **Agent Builder** (Cloud) | In-browser prototyping + deploy, no code |

## Minimal Python Agent (shape)
```python
from livekit.agents import Agent, AgentSession, JobContext

async def entrypoint(ctx: JobContext) -> None:
    await ctx.connect()
    agent = Agent(instructions="You are a helpful voice assistant.")
    session = AgentSession()
    await session.start(agent=agent, room=ctx.room)
```

## Topics To Explore Next
- Multimodality (vision + audio + text fusion)
- Logic / workflow structure (state, tools, handoff)
- Agent server management (worker pools, scaling)
- Available models and provider integrations

## Tags
[[LiveKit]] [[AI Agents]] [[Voice AI]] [[Multimodal]] [[Python]] [[Agent Builder]]
