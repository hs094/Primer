# 18 — Testing

🔑 Drive `AgentSession` with a text-mode LLM under pytest; use `result.expect.next_event().is_message(...).judge(llm, intent=...)` to grade behavior cheaply.

Source: https://docs.livekit.io/agents/build/testing/

## What to cover
- Expected behavior — intent and tone for typical inputs
- Tool usage — correct function called with correct args
- Error handling — invalid inputs, tool failures
- Grounding — factual accuracy, no hallucination
- Misuse resistance — manipulation attempts

## Pytest pattern

```python
import pytest
from livekit.agents import AgentSession, inference

@pytest.mark.asyncio
async def test_assistant_greeting() -> None:
    async with (
        inference.LLM(model="openai/gpt-5.3-chat-latest") as llm,
        AgentSession(llm=llm) as session,
    ):
        await session.start(Assistant())
        result = await session.run(user_input="Hello")
        await result.expect.next_event().is_message(role="assistant").judge(
            llm, intent="Makes a friendly introduction and offers assistance.",
        )
```

The built-in helpers operate on text I/O — the LLM (via LiveKit Inference or a plugin in text-only mode) is the only paid call, making behavioral coverage cheap.

## Caveats
- `get_job_context()` raises `RuntimeError` outside a real job — patch with `unittest.mock` if your code touches it.
- Built-in helpers do **not** exercise the audio pipeline. For full STT/TTS coverage, use third-party services (Bluejay, Cekura, Coval, Hamming).

## Assertion surface
`result.expect` exposes a fluent chain over emitted events:
- `.next_event()` — pop next
- `.is_message(role=...)` — assert event is a chat message
- `.is_function_call(name=...)` — assert tool call
- `.judge(llm, intent=...)` — LLM-graded check on content

## Tags
[[LiveKit]] [[Agents]] [[Testing]] [[pytest]] [[Evaluation]]
