# CO 06 — Memory

🔑 Short-term memory = thread-scoped checkpoints; long-term memory = cross-thread store entries — different APIs, different lifetimes, both needed for real agents.

## The two layers

| Memory | Scope | Backed by | API |
|--------|-------|-----------|-----|
| Short-term | One conversation thread | Checkpointer | `compile(checkpointer=...)`, `thread_id` |
| Long-term | Across threads / users | Store | `compile(store=...)`, namespaces |

See [[LangGraph CO 05 - Persistence (Checkpointers)]] for the storage primitives.

## Short-term: checkpointer + `thread_id`

```python
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.graph import StateGraph

checkpointer = InMemorySaver()
builder = StateGraph(...)
graph = builder.compile(checkpointer=checkpointer)

graph.invoke(
    {"messages": [{"role": "user", "content": "hi! i am Bob"}]},
    {"configurable": {"thread_id": "1"}},
)
```

Reuse the same `thread_id` to continue the conversation; pick a new one to start fresh.

### Production backends

- `langgraph.checkpoint.postgres.PostgresSaver` (+ `AsyncPostgresSaver`)
- `langgraph.checkpoint.mongodb.MongoDBSaver`
- `langgraph.checkpoint.redis.RedisSaver`

All require `checkpointer.setup()` on first use.

### Subgraphs

Checkpointers compiled on a parent automatically propagate to subgraphs. A subgraph can opt for its own scoping with `checkpointer=True` at compile time.

## Long-term: store + namespaces

```python
from langgraph.store.memory import InMemoryStore
from langgraph.graph import StateGraph

store = InMemoryStore()
builder = StateGraph(...)
graph = builder.compile(store=store)
```

Inside a node, access via the injected `Runtime`:

```python
from langgraph.runtime import Runtime

async def call_model(state: MessagesState, runtime: Runtime[Context]):
    user_id = runtime.context.user_id
    namespace = (user_id, "memories")

    memories = await runtime.store.asearch(
        namespace, query=state["messages"][-1].content, limit=3
    )

    await runtime.store.aput(
        namespace, str(uuid.uuid4()), {"data": "User prefers dark mode"}
    )
```

Namespace is a tuple — `(user_id, "memories")`, `(org, user, "facts")`. Pick a hierarchy that matches how you'll query.

Production: Postgres, Redis, MongoDB stores — sync and async variants, all need `store.setup()`.

## Semantic search

Configure embeddings on the store to search by similarity instead of exact match:

```python
from langchain.embeddings import init_embeddings
from langgraph.store.memory import InMemoryStore

embeddings = init_embeddings("openai:text-embedding-3-small")
store = InMemoryStore(
    index={"embed": embeddings, "dims": 1536}
)

store.put(("user_123", "memories"), "1", {"text": "I love pizza"})
items = store.search(
    ("user_123", "memories"), query="I'm hungry", limit=1
)
```

## Managing message history

Long threads blow up context windows. Three strategies:

### Trim by tokens

```python
from langchain_core.messages.utils import trim_messages, count_tokens_approximately

messages = trim_messages(
    state["messages"],
    strategy="last",
    token_counter=count_tokens_approximately,
    max_tokens=128,
    start_on="human",
    end_on=("human", "tool"),
)
```

### Delete with `RemoveMessage`

```python
from langchain.messages import RemoveMessage

def delete_messages(state):
    if len(state["messages"]) > 2:
        return {"messages": [RemoveMessage(id=m.id) for m in state["messages"][:2]]}
```

Wipe everything: `RemoveMessage(id=REMOVE_ALL_MESSAGES)`.

### Summarize

Extend `MessagesState` with a `summary` field; periodically fold older messages into it.

```python
from langgraph.graph import MessagesState

class State(MessagesState):
    summary: str

def summarize_conversation(state: State):
    summary = state.get("summary", "")
    summary_message = (
        f"Summary: {summary}\nExtend this summary:" if summary
        else "Create a summary:"
    )
    messages = state["messages"] + [HumanMessage(content=summary_message)]
    response = model.invoke(messages)
    delete_messages = [RemoveMessage(id=m.id) for m in state["messages"][:-2]]
    return {"summary": response.content, "messages": delete_messages}
```

Run this conditionally — every N turns or above a token threshold.

## Inspecting and clearing

```python
config = {"configurable": {"thread_id": "1"}}

graph.get_state(config)               # current snapshot
list(graph.get_state_history(config)) # all checkpoints, newest first

checkpointer.delete_thread("1")       # wipe the thread
```

## Wiring both

```python
graph = builder.compile(checkpointer=checkpointer, store=store)
```

A typical agent reads short-term context from the checkpointer (it's automatic, just use the same `thread_id`) and pulls personalized facts from the store via `runtime.store` inside the model node.

## Key terms

- **`thread_id`** — identifies one conversation; switches short-term memory.
- **Namespace** — tuple key for long-term memory, e.g. `(user_id, "memories")`.
- **Context schema** — typed shape of `runtime.context` (e.g. `Context(user_id=...)`).
- **Reducer** — controls how state updates merge; `add_messages` appends to the message list.

⚠️ A checkpointer is bound to the graph at compile time; the `thread_id` is per call. Mixing those up is the most common "why doesn't memory work" bug.

⚠️ `runtime.store` is `None` if you didn't pass `store=` to `compile()`. Guard or wire it up.

💡 Short-term memory is plumbing — wire it once and forget it. Long-term memory is product design — what you choose to remember (and how you namespace it) defines the agent's behavior across sessions.
