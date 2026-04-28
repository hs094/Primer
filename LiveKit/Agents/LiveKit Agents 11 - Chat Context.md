# 11 — Chat Context

🔑 `ChatContext` is the ordered list of items (messages, tool calls, handoffs) sent to the LLM each turn — copy/truncate/merge it explicitly to control what the model sees.

Source: https://docs.livekit.io/agents/logic/chat-context/

## Accessing
Within an `Agent` or task, use `self.chat_ctx`. Session-wide history lives on `session.history`.

```python
class MyAgent(Agent):
    async def on_enter(self) -> None:
        print(self.chat_ctx.items)
```

## Item types
Each item has a `type`:
- `message` — role (`system` / `user` / `assistant`) + content
- `function_call` — LLM-requested tool invocation
- `function_call_output` — tool result
- `agent_handoff` — automatic marker on handoff
- `agent_config_update` — instruction/tool change (Python only)

Read text via `item.text_content`.

## Operations

```python
from livekit.agents import ChatContext

chat_ctx = ChatContext()
chat_ctx.add_message(role="system", content="You are a helpful assistant.")

# Copy with filters
full_copy = self.chat_ctx.copy()
turns_only = self.chat_ctx.copy(
    exclude_instructions=True,
    exclude_function_call=True,
)

# Truncate to last N items
recent = self.chat_ctx.copy().truncate(max_items=6)

# Merge another context in
primary_ctx.merge(other_ctx, exclude_function_call=True)
```

## Seeding a session
```python
initial_ctx = ChatContext()
initial_ctx.add_message(role="assistant", content=f"The user's name is {user_name}.")
await session.start(room=ctx.room, agent=MyAgent(chat_ctx=initial_ctx))
```

## RAG inside `on_user_turn_completed`
```python
async def on_user_turn_completed(
    self, turn_ctx: ChatContext, new_message: ChatMessage,
) -> None:
    extra = await fetch_relevant_data(new_message.text_content)
    turn_ctx.add_message(role="assistant", content=extra)
    await self.update_chat_ctx(turn_ctx)
```

## Images
```python
from livekit.agents.llm import ImageContent

initial_ctx.add_message(
    role="user",
    content=["Here is a picture of me", ImageContent(image="https://example.com/image.jpg")],
)
```

## Tags
[[LiveKit]] [[Agents]] [[ChatContext]] [[ChatMessage]] [[RAG]]
