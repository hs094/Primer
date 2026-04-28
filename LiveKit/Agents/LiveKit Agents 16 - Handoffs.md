# 16 — Handoffs

🔑 Return a new `Agent` instance from a `@function_tool` to hand off; pass `chat_ctx=self.chat_ctx.copy(exclude_instructions=True)` to carry conversation but drop the old system prompt.

Source: https://docs.livekit.io/agents/logic/agents-handoffs/

## Defining an agent

```python
class HelpfulAssistant(Agent):
    def __init__(self) -> None:
        super().__init__(instructions="You are a helpful voice AI assistant.")
```

The active agent owns the session. Swap via `session.update_agent(NewAgent())` or — cleaner — return one from a tool.

## Handoff via tool return

```python
@function_tool()
async def transfer_to_billing(self, context: RunContext):
    """Transfer the customer to a billing specialist."""
    return BillingAgent(chat_ctx=self.chat_ctx), "Transferring to billing"
```

A tuple `(Agent, str)` returns the spoken handoff message alongside the new agent. An `AgentHandoff` item is added to chat context recording old/new agent IDs.

## Context preservation
By default the new agent starts with empty history.

```python
return TechnicalSupportAgent(
    chat_ctx=self.chat_ctx.copy(exclude_instructions=True),
)
```

For long conversations, summarize before handoff (separate LLM call) and seed the new agent with the summary instead of the full transcript.

## Shared session state
Use `userdata` on the session for typed state shared across agents:

```python
from dataclasses import dataclass

@dataclass
class MySessionInfo:
    customer_id: str | None = None

session = AgentSession[MySessionInfo](userdata=MySessionInfo(), ...)
```

Read in tools via `context.userdata`, in agents via `self.session.userdata`.

## Tags
[[LiveKit]] [[Agents]] [[Handoffs]] [[function_tool]] [[Userdata]]
