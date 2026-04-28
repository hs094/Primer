# 12 — Tasks and Function Tools

🔑 `@function_tool` exposes a Python coroutine to the LLM; it receives a `RunContext` and can return a value, speak, hand off, or call frontend RPC.

Source: https://docs.livekit.io/agents/build/tools/

## Two flavors
- **Function tools** — your code, executed in the agent process.
- **Provider tools** — server-side built-ins from the LLM provider (single API round-trip).

## Function tool

```python
from livekit.agents import Agent, RunContext, function_tool

class Assistant(Agent):
    @function_tool()
    async def lookup_order(self, context: RunContext, order_id: str) -> dict:
        """Look up an order by ID.

        Args:
            order_id: The customer's order identifier.
        """
        return await db.fetch_order(order_id)
```

The docstring becomes the tool description; type-annotated args become the JSON schema. From a tool you can:
- return a value (becomes `function_call_output`)
- `await context.session.say(...)` for inline speech
- return a new `Agent` instance to hand off
- call `context.session.room.local_participant.perform_rpc(...)` to invoke frontend methods

## Provider tools

```python
from livekit.plugins import openai

agent = MyAgent(
    llm=openai.responses.LLM(model="gpt-4.1"),
    tools=[openai.tools.WebSearch()],
)
```

Available: OpenAI `WebSearch` / `FileSearch` / `CodeInterpreter`; Gemini `GoogleSearch` / `GoogleMaps` / `URLContext` / `FileSearch` / `ToolCodeExecution`; Anthropic `ComputerUse`; xAI `WebSearch` / `XSearch` / `FileSearch`.

## Tasks
Tasks are short-lived units of work that run to completion and return a typed result — use them for discrete sub-flows (consent collection, verification) where an `Agent`'s long-lived session control is overkill.

## Tags
[[LiveKit]] [[Agents]] [[function_tool]] [[RunContext]] [[Tools]] [[MCP]]
