# 02 — Voice AI Quickstart

🔑 `lk agent init` scaffolds a working voice agent in under 10 minutes — pick STT-LLM-TTS or a realtime model, run it in `console`/`dev`, deploy with `lk agent create`.

Source: https://docs.livekit.io/agents/start/voice-ai-quickstart/

## Prerequisites
- Python 3.10+ with `uv`, or Node 20+ with `pnpm` >= 10.15.0
- LiveKit Cloud account for API key/secret and agent hosting

## Scaffold
```bash
lk agent init my-agent --template agent-starter-python
```
Clones the starter, writes `.env.local` with credentials, prints next steps.

## STT-LLM-TTS pipeline (default starter)
```python
from livekit.agents import Agent, AgentSession, JobContext, WorkerOptions, cli
from livekit.plugins import deepgram, openai, cartesia, silero

class Assistant(Agent):
    def __init__(self) -> None:
        super().__init__(instructions="You are a helpful voice assistant.")

async def entrypoint(ctx: JobContext) -> None:
    session = AgentSession(
        stt=deepgram.STT(model="nova-3"),
        llm=openai.LLM(model="gpt-5.3-chat"),
        tts=cartesia.TTS(model="sonic-3"),
        vad=silero.VAD.load(),
    )
    await session.start(agent=Assistant(), room=ctx.room)
    await session.generate_reply(instructions="Greet the user.")

if __name__ == "__main__":
    cli.run_app(WorkerOptions(entrypoint_fnc=entrypoint))
```

## Realtime model variant
```python
from livekit.plugins import openai

session = AgentSession(llm=openai.realtime.RealtimeModel())
```

## Run modes
- `python agent.py console` — local terminal loop (Python only)
- `python agent.py dev` — connects to LiveKit Cloud, hot reload, debug
- `python agent.py start` — production worker

## Deploy
`lk agent create` packages and deploys to LiveKit Cloud. Frontends connect via web/mobile SDKs or SIP for telephony.

## Tags
[[LiveKit]] [[Agents]] [[Quickstart]] [[Deepgram]] [[Cartesia]] [[OpenAI]]
