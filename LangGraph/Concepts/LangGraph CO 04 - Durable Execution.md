# CO 04 — Durable Execution

🔑 Pause and resume from a checkpoint without re-running completed work — but only if your nodes are deterministic and side effects are wrapped in tasks.

## The contract

To get durability, three things must hold:

1. A **checkpointer** is attached to the compiled graph (see [[LangGraph CO 05 - Persistence (Checkpointers)]]).
2. A **`thread_id`** is passed in the run config — it identifies the workflow instance.
3. **Non-deterministic and side-effecting work is wrapped in `@task`** so it's recorded once and replayed from the checkpoint, not re-executed.

## How resume works

Resumption does *not* continue from the exact line where execution stopped. Instead, the runtime picks an appropriate restart point:

- **StateGraph**: restart at the beginning of the node that was running.
- **Functional API**: restart at the entrypoint.
- **Subgraphs**: restart at the parent node that called the subgraph.

Anything that already ran inside that node *will run again* — unless it was inside a `@task`, whose result was checkpointed.

## Wrap side effects in `@task`

```python
from langgraph.func import task
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.graph import StateGraph, START, END

@task
def _make_request(url: str):
    """Make a request."""
    return requests.get(url).text[:100]

def call_api(state: State):
    """Example node that makes an API request."""
    requests = [_make_request(url) for url in state['urls']]
    results = [request.result() for request in requests]
    return {"results": results}

builder = StateGraph(State)
builder.add_node("call_api", call_api)
builder.add_edge(START, "call_api")
builder.add_edge("call_api", END)

checkpointer = InMemorySaver()
graph = builder.compile(checkpointer=checkpointer)

config = {"configurable": {"thread_id": thread_id}}
graph.invoke({"urls": ["https://www.example.com"]}, config)
```

Tasks return futures; `.result()` resolves them. On replay, `_make_request` returns its previously checkpointed value instead of hitting the network again.

## Rules for what to wrap

Wrap inside `@task` (or a dedicated node):

- Non-deterministic computations — `random`, `uuid4`, `datetime.now()`.
- External calls — HTTP, DB writes, file I/O.
- Anything with observable side effects — logging that must not duplicate, payment calls, email sends.

If a node performs **multiple** side effects, give each its own `@task`. Two side effects in the same task replay as one atomic unit, which is rarely what you want.

Make tasks **idempotent** wherever possible — replay can still happen if a checkpoint write fails after the task ran but before it was persisted.

## Durability modes

Trade-off between performance and recovery granularity. Pass `durability=` on `invoke`/`stream`:

```python
graph.stream({"input": "test"}, durability="sync")
```

| Mode | Behavior | Use when |
|------|----------|----------|
| `"exit"` | Persist only at completion. | Throughput matters, mid-run recovery doesn't. |
| `"async"` | Checkpoint asynchronously during the next step. | Default trade-off; small data-loss window on crash. |
| `"sync"` | Checkpoint synchronously before each step advances. | Strict recovery requirements; pays per-step latency. |

## Resumption scenarios

### Human-in-the-loop pause

Use `interrupt()` inside a node and resume with a `Command`:

```python
from langgraph.types import interrupt, Command

def review(state: State):
    decision = interrupt({"prompt": "approve?", "draft": state["draft"]})
    return {"approved": decision}

# resume after the interrupt
graph.invoke(Command(resume="yes"), config)
```

### Failure recovery

After a crash, re-invoke with the same `thread_id` and `None` input — the runtime picks up from the last checkpoint:

```python
graph.invoke(None, {"configurable": {"thread_id": thread_id}})
```

## Determinism gotchas

- A node that picks a random branch on each call will diverge on replay — wrap the random draw in a `@task`.
- A node that reads `time.time()` to pick a code path will replay differently — checkpoint the timestamp.
- Streaming partial output to the user is a side effect; if exact replay matters, gate it behind a task or accept that streamed bytes may repeat.

⚠️ Two side effects, one node, no tasks = double execution on resume. The runtime cannot know which side effect already happened.

💡 Treat every node as "may run more than once" by default. The cure is `@task` wrappers around the parts that must run exactly once. See [[LangGraph CO 05 - Persistence (Checkpointers)]] for the storage layer that makes this work.
