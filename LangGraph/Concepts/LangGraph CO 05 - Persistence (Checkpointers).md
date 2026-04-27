# CO 05 — Persistence (Checkpointers)

🔑 A checkpointer snapshots graph state at every super-step, keyed by `thread_id` — that's what makes memory, time-travel, human-in-the-loop, and crash recovery work.

## The model

- **Thread** — a `thread_id` string identifying one workflow instance. Every checkpoint belongs to exactly one thread.
- **Checkpoint** — a `StateSnapshot` taken at a super-step boundary. For `START → A → B → END` you get four: empty/START-next, A-next, B-next, no-next.
- **Super-step** — one tick of the [[LangGraph CO 03 - Pregel Runtime]]; all scheduled nodes run, then a checkpoint is written.

Pass the thread on every call:

```python
config = {"configurable": {"thread_id": "1"}}
graph.invoke({...}, config)
```

## `StateSnapshot` fields

| Field | Type | Meaning |
|-------|------|---------|
| `values` | `dict` | Channel values at this checkpoint. |
| `next` | `tuple[str, ...]` | Nodes scheduled to run next. |
| `config` | `dict` | `thread_id`, `checkpoint_ns`, `checkpoint_id`. |
| `metadata` | `dict` | `source`, `writes`, `step`. |
| `created_at` | `str` | ISO 8601 timestamp. |
| `parent_config` | `dict \| None` | Previous checkpoint config; `None` at root. |
| `tasks` | `tuple` | Tasks executing at this step. |

## Read APIs

```python
config = {"configurable": {"thread_id": "1"}}

# latest
graph.get_state(config)

# specific checkpoint
graph.get_state({"configurable": {"thread_id": "1", "checkpoint_id": "..."}})

# full history, newest first
list(graph.get_state_history(config))
```

Filter the history with regular Python:

```python
history = list(graph.get_state_history(config))
before_node_b = next(s for s in history if s.next == ("node_b",))
step_2 = next(s for s in history if s.metadata["step"] == 2)
```

## Update / fork

`update_state` writes new values *through reducers* and creates a new checkpoint that branches off the current one. `as_node` controls which node the writes appear to come from:

```python
graph.update_state(config, {"key": "value"}, as_node="node_name")
```

Time-travel: pass an older `checkpoint_id` to invoke from there. Nodes before that point are skipped; nodes after re-execute on the new branch.

## Checkpointer implementations

| Class | Package | Use |
|-------|---------|-----|
| `InMemorySaver` | `langgraph` (built-in) | Notebooks, tests. State dies with the process. |
| `SqliteSaver` / `AsyncSqliteSaver` | `langgraph-checkpoint-sqlite` | Local single-process workflows. |
| `PostgresSaver` / `AsyncPostgresSaver` | `langgraph-checkpoint-postgres` | Production. |
| `CosmosDBSaver` / `AsyncCosmosDBSaver` | `langgraph-checkpoint-cosmosdb` | Azure-native. |

All implement `BaseCheckpointSaver` — `.put()`, `.put_writes()`, `.get_tuple()`, `.list()` (plus `aput`, `aget_tuple`, etc.).

### Postgres

```python
from langgraph.checkpoint.postgres import PostgresSaver

checkpointer = PostgresSaver.from_conn_string("postgresql://...")
checkpointer.setup()                                # one-time schema migration

graph = builder.compile(checkpointer=checkpointer)
```

`setup()` is idempotent but expensive — call it during deploy/startup, not per request. Async variants (`AsyncPostgresSaver`) follow the same shape with `await`.

### In-memory for tests

```python
from langgraph.checkpoint.memory import InMemorySaver
checkpointer = InMemorySaver()
graph = builder.compile(checkpointer=checkpointer)
```

## Serialization

Default is `JsonPlusSerializer` — handles LangChain primitives, `datetime`, `Enum`, Pydantic.

Pickle fallback for arbitrary Python objects (use sparingly):

```python
from langgraph.checkpoint.serde.jsonplus import JsonPlusSerializer

graph.compile(
    checkpointer=InMemorySaver(serde=JsonPlusSerializer(pickle_fallback=True))
)
```

Encrypt at rest with AES:

```python
from langgraph.checkpoint.serde.encrypted import EncryptedSerializer
from langgraph.checkpoint.sqlite import SqliteSaver

serde = EncryptedSerializer.from_pycryptodome_aes()   # reads key from env
checkpointer = SqliteSaver(sqlite3.connect("checkpoint.db"), serde=serde)
```

## Store — cross-thread state

A checkpointer is per-thread. The `Store` interface is *across* threads — same user across many conversations, shared knowledge, etc.

```python
from langgraph.store.memory import InMemoryStore

store = InMemoryStore()

namespace = (user_id, "memories")
memory_id = str(uuid.uuid4())
store.put(namespace, memory_id, {"food_preference": "I like pizza"})

memories = store.search(namespace)
memories[-1].dict()
```

Namespaces are tuples — group by user, by domain, or both: `(user_id, "memories")`, `(org_id, user_id, "facts")`.

### Semantic search

Index entries with embeddings to retrieve by similarity:

```python
from langchain.embeddings import init_embeddings

store = InMemoryStore(
    index={
        "embed": init_embeddings("openai:text-embedding-3-small"),
        "dims": 1536,
        "fields": ["food_preference", "$"],
    }
)

store.put(
    namespace,
    str(uuid.uuid4()),
    {"food_preference": "I love Italian cuisine", "context": "Dinner plans"},
    index=["food_preference"],
)
```

`fields` is the default index spec; per-`put` `index=` overrides it.

Production stores: Postgres, Redis, MongoDB — same API, async variants, all need `setup()` once.

## Wiring it all together

```python
graph = builder.compile(checkpointer=checkpointer, store=store)
```

Checkpointer handles the conversation thread; store handles long-lived facts. See [[LangGraph CO 06 - Memory]] for how nodes consume the store via `Runtime`.

⚠️ A checkpointer compiles into the graph. Forgetting to pass `thread_id` at invoke time means each call starts fresh — the checkpointer is silently useless.

⚠️ `update_state` runs writes through the same reducers as normal node output. A `LastValue` channel will overwrite; an `add_messages` channel will append.

💡 Pick the smallest checkpointer that survives your failure domain. In-memory for tests, SQLite for single-box, Postgres for anything that must outlive a deploy.
