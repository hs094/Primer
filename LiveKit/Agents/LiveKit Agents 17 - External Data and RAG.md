# 17 — External Data and RAG

🔑 Three injection points for external data: prewarm/job metadata for static context, `on_user_turn_completed` for inline RAG, and `@function_tool` for precise on-demand calls.

Source: https://docs.livekit.io/agents/build/external-data/

## Initial context
Load before the session starts; pass via job metadata, room metadata, or participant attributes. Seed the agent with a `ChatContext`:

```python
initial_ctx = ChatContext()
initial_ctx.add_message(role="assistant", content=f"User profile: {profile_json}")
await session.start(room=ctx.room, agent=MyAgent(chat_ctx=initial_ctx))
```

Use a prewarm function for static assets (embeddings, indexes); only put per-user data through job metadata so the entrypoint does not block on network.

## Inline RAG via `on_user_turn_completed`
Runs after user input ends but before LLM inference — no tool-call round-trip.

```python
async def on_user_turn_completed(
    self, turn_ctx: ChatContext, new_message: ChatMessage,
) -> None:
    docs = await vector_store.search(new_message.text_content, k=4)
    turn_ctx.add_message(
        role="assistant",
        content=f"Relevant context:\n{docs}",
    )
    await self.update_chat_ctx(turn_ctx)
```

## Tool calls for actions
Define focused tools (`search_calendar`, `create_event`) when you need precision or side effects. Pass credentials through participant attributes / job metadata, not hardcoded.

## UX during long lookups
- **Verbal status**: `await session.say("Let me check that for you.")`; cancel if the operation finishes quickly.
- **Thinking sounds**: background audio with `BuiltinAudioClip.KEYBOARD_TYPING`.
- **Frontend UI**: RPC calls to show cancellable popups / progress.

## Memory / KB integrations
Letta (cross-session memory), LlamaIndex (KB framework), Mem0 (self-improving memory), AgentMail (email inbox).

## Tags
[[LiveKit]] [[Agents]] [[RAG]] [[ExternalData]] [[ChatContext]]
