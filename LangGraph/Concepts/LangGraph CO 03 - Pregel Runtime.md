# CO 03 — Pregel Runtime

🔑 Every LangGraph app is a `Pregel` instance — actors (nodes) communicate via channels in repeating BSP super-steps; `StateGraph` and `@entrypoint` are sugar over this.

## Bulk Synchronous Parallel model

A super-step is one tick of the engine. Three phases per tick:

1. **Plan** — pick which actors run. On step 0, that's anything subscribed to an input channel; afterwards, anything subscribed to a channel that was just written.
2. **Execution** — run all selected actors in parallel until completion, failure, or timeout.
3. **Update** — apply each actor's channel writes; these become the inputs the next plan phase sees.

Loops until no actor is selected, an interrupt fires, or `recursion_limit` is reached.

## Actors and channels

- **Actor** — a `PregelNode` that subscribes to channels, runs a callable, writes to other channels. Implements LangChain's `Runnable`.
- **Channel** — typed mailbox between actors. Three properties: value type, update type, update function.

### Built-in channels

- `LastValue(t)` — default; holds the last write. Used for normal state keys.
- `EphemeralValue(t)` — value visible only in the current super-step; not persisted.
- `Topic(t, accumulate=True)` — pub/sub; collects all writes in a step, optionally accumulates across steps.
- `BinaryOperatorAggregate(t, operator=fn)` — folds writes through a reducer (think `operator.add`).

## Direct Pregel construction

The lowest-level API. Most users skip it, but reading it clarifies the model.

```python
from langgraph.channels import EphemeralValue
from langgraph.pregel import Pregel, NodeBuilder

node1 = (
    NodeBuilder().subscribe_only("a")
    .do(lambda x: x + x)
    .write_to("b")
)

app = Pregel(
    nodes={"node1": node1},
    channels={
        "a": EphemeralValue(str),
        "b": EphemeralValue(str),
    },
    input_channels=["a"],
    output_channels=["b"],
)

app.invoke({"a": "foo"})
```

### Multi-node chain

```python
node1 = NodeBuilder().subscribe_only("a").do(lambda x: x + x).write_to("b")
node2 = NodeBuilder().subscribe_only("b").do(lambda x: x + x).write_to("c")

app = Pregel(
    nodes={"node1": node1, "node2": node2},
    channels={
        "a": EphemeralValue(str),
        "b": LastValue(str),
        "c": EphemeralValue(str),
    },
    input_channels=["a"],
    output_channels=["b", "c"],
)
```

`b` is `LastValue` so node2's plan-phase subscription sees the persisted write.

### Topic for fan-in

```python
from langgraph.channels import Topic

channels={
    "a": EphemeralValue(str),
    "b": EphemeralValue(str),
    "c": Topic(str, accumulate=True),
}
```

Multiple writers feed `c`; `accumulate=True` keeps prior writes across steps.

### Reducer aggregation

```python
from langgraph.channels import BinaryOperatorAggregate

def reducer(current, update):
    if current:
        return current + " | " + update
    return update

channels={
    "c": BinaryOperatorAggregate(str, operator=reducer),
}
```

### Cycles

A node that writes back to a channel it subscribes to forms a loop. Use `ChannelWriteEntry(..., skip_none=True)` to terminate when the node returns `None`.

```python
from langgraph.pregel import ChannelWriteEntry

example_node = (
    NodeBuilder().subscribe_only("value")
    .do(lambda x: x + x if len(x) < 10 else None)
    .write_to(ChannelWriteEntry("value", skip_none=True))
)
```

## StateGraph compiles to Pregel

```python
from langgraph.graph import StateGraph
from langgraph.constants import START

class Essay(TypedDict):
    topic: str
    content: str | None
    score: float | None

builder = StateGraph(Essay)
builder.add_node(write_essay)
builder.add_node(score_essay)
builder.add_edge(START, "write_essay")
builder.add_edge("write_essay", "score_essay")

graph = builder.compile()
print(graph.nodes)
print(graph.channels)
```

`graph.nodes` and `graph.channels` expose the underlying actors and channels — useful for inspection and debugging.

## Functional API also compiles to Pregel

```python
from langgraph.func import entrypoint
from langgraph.checkpoint.memory import InMemorySaver

checkpointer = InMemorySaver()

@entrypoint(checkpointer=checkpointer)
def write_essay(essay: Essay):
    return {"content": f"Essay about {essay['topic']}"}

print(write_essay.nodes)
print(write_essay.channels)
```

## Why this matters

- **Determinism per super-step** — within a step, actors run in parallel but observe a consistent channel snapshot. No mid-step writes leak.
- **Replay** — channel writes are checkpointable, which is what makes [[LangGraph CO 04 - Durable Execution]] and [[LangGraph CO 05 - Persistence (Checkpointers)]] possible.
- **Fan-out via `Send`** — orchestrator-worker patterns ([[LangGraph CO 01 - Workflows and Agents]]) work because workers can be scheduled into the next super-step dynamically.

⚠️ State channels default to `LastValue`. If you fan-in from parallel nodes without a reducer-typed key, the second writer's value silently overwrites the first. Annotate with `Annotated[list, operator.add]` (or similar) to accumulate.

💡 Most code lives at the StateGraph level, but knowing the Pregel layer makes parallelism, channel reducers, and checkpoint replay obvious instead of magic.
