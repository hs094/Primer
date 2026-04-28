# 14 — Pipeline Nodes and Hooks

🔑 Override pipeline nodes (`stt_node`, `llm_node`, `tts_node`, `transcription_node`) on an `Agent` subclass and call `Agent.default.<node>()` inside to wrap default behavior.

Source: https://docs.livekit.io/agents/build/nodes/

## Lifecycle hooks

```python
class Greeter(Agent):
    async def on_enter(self) -> None:
        await self.session.generate_reply(
            instructions="Greet the user with a warm welcome",
        )

    async def on_exit(self) -> None:
        await self.session.say("Transferring you now.")

    async def on_user_turn_completed(
        self, turn_ctx: ChatContext, new_message: ChatMessage,
    ) -> None:
        # RAG / filtering before the LLM sees the turn
        ...
```

- `on_enter` — agent becomes active.
- `on_exit` — control about to transfer away.
- `on_user_turn_completed` — user input ended, before LLM call.

## Pipeline nodes (STT-LLM-TTS)

- **`stt_node`** — audio frames -> speech events. Customize for preprocessing, buffering, alternate STT.
- **`llm_node`** — runs inference on `chat_ctx`; yields `str` chunks or `ChatChunk` objects (for tool calls / metadata).
- **`tts_node`** — text -> audio. Override for chunking, custom engines, post-FX.
- **`transcription_node`** — post-processes transcripts (punctuation, formatting, timestamp alignment).

## Realtime model nodes
- **`realtime_audio_output_node`** — modifies audio before it leaves to the user (volume, FX).

## Override pattern

```python
class CustomAgent(Agent):
    async def llm_node(self, chat_ctx, tools, model_settings):
        # pre-processing
        async for chunk in Agent.default.llm_node(self, chat_ctx, tools, model_settings):
            # transform chunks here
            yield chunk
```

## Tags
[[LiveKit]] [[Agents]] [[Pipeline]] [[Hooks]] [[Nodes]]
