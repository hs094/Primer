# PT 03 — Time Travel

🔑 Checkpoints turn a thread into a navigable history — `get_state_history` lists every super-step, replay re-runs from any of them, and `update_state` forks a new branch with edited state.

## Mental Model

Time travel needs a checkpointer (see [[LangGraph CO 05 - Persistence (Checkpointers)]]) and a stable `thread_id`. Two operations:

- **Replay** — invoke from a prior checkpoint config; nodes after that point re-execute.
- **Fork** — call `update_state(prior_config, values=...)`; LangGraph writes a new checkpoint branched off the old one and returns its config.

`update_state` does **not** rewind the thread. It creates a new branch. Replay re-runs nodes — LLM calls, API hits, and `interrupt()` fire again and may produce different results.

## Listing History

```python
from langgraph.graph import StateGraph, START
from langgraph.checkpoint.memory import InMemorySaver
from typing_extensions import TypedDict, NotRequired
from langchain_core.utils.uuid import uuid7

class State(TypedDict):
    topic: NotRequired[str]
    joke: NotRequired[str]

def generate_topic(state): return {"topic": "socks in the dryer"}
def write_joke(state): return {"joke": f"Why do {state['topic']} disappear? They elope!"}

checkpointer = InMemorySaver()
graph = (
    StateGraph(State)
    .add_node("generate_topic", generate_topic)
    .add_node("write_joke", write_joke)
    .add_edge(START, "generate_topic")
    .add_edge("generate_topic", "write_joke")
    .compile(checkpointer=checkpointer)
)

config = {"configurable": {"thread_id": str(uuid7())}}
result = graph.invoke({}, config)

history = list(graph.get_state_history(config))
for state in history:
    print(f"next={state.next}, checkpoint_id={state.config['configurable']['checkpoint_id']}")
```

`get_state_history` returns `StateSnapshot` objects in **reverse chronological** order. Each snapshot exposes `.next` (the nodes due to run from that checkpoint), `.values` (state at that point), and `.config` (the dict you pass back to replay).

## Replay

Pick a snapshot and invoke with `None`:

```python
before_joke = next(s for s in history if s.next == ("write_joke",))

replay_result = graph.invoke(None, before_joke.config)
# write_joke re-executes; generate_topic does not
```

The replay config carries `{"configurable": {"thread_id": "...", "checkpoint_id": "..."}}` — LangGraph uses the `checkpoint_id` to start from that exact super-step.

## Fork via `update_state`

```python
before_joke = next(s for s in history if s.next == ("write_joke",))

fork_config = graph.update_state(
    before_joke.config,
    values={"topic": "chickens"},
)

fork_result = graph.invoke(None, fork_config)
print(fork_result["joke"])  # joke about chickens
```

`update_state` returns a fresh config pointing at the new branch. Pass it to `invoke()` to continue from there.

### `as_node`

Tell LangGraph which node "wrote" the update so it can route correctly:

```python
fork_config = graph.update_state(
    before_joke.config,
    values={"topic": "chickens"},
    as_node="generate_topic",
)
# resumes at write_joke (successor of generate_topic)
```

Use `as_node` when parallel branches updated state, when there's no execution history yet, or when you want to skip past a node intentionally.

## Time Travel Through Interrupts

Replay re-fires `interrupt()` calls. The first invoke pauses again — feed a fresh `Command(resume=...)`:

```python
from langgraph.types import interrupt, Command

def ask_human(state):
    answer = interrupt("What is your name?")
    return {"value": [f"Hello, {answer}!"]}

# initial run
graph.invoke({"value": []}, config)
graph.invoke(Command(resume="Alice"), config)

# replay from before ask_human
history = list(graph.get_state_history(config))
before_ask = [s for s in history if s.next == ("ask_human",)][-1]

graph.invoke(None, before_ask.config)               # pauses at interrupt
graph.invoke(Command(resume="Bob"), before_ask.config)  # different answer

# fork with edited state, then re-answer
fork_config = graph.update_state(before_ask.config, {"value": ["forked"]})
graph.invoke(None, fork_config)
graph.invoke(Command(resume="Bob"), fork_config)
# {"value": ["forked", "Hello, Bob!", "Done"]}
```

### Selective Forking Between Interrupts

Fork after the first interrupt but before the second to keep early answers and re-collect later ones:

```python
history = list(graph.get_state_history(config))
between = [s for s in history if s.next == ("ask_age",)][-1]

fork_config = graph.update_state(between.config, {"value": ["modified"]})
result = graph.invoke(None, fork_config)
# ask_name's answer preserved; ask_age pauses for a new resume
```

## Subgraph Time Travel

By default, a subgraph compiled without its own checkpointer inherits the parent's, and the parent treats the entire subgraph as a single super-step — you can only fork from before the subgraph node.

For finer granularity, compile the subgraph with `checkpointer=True` so it keeps internal checkpoints:

```python
subgraph = (
    StateGraph(State)
    .add_node("step_a", step_a)
    .add_node("step_b", step_b)
    .add_edge(START, "step_a")
    .add_edge("step_a", "step_b")
    .compile(checkpointer=True)
)

graph = (
    StateGraph(State)
    .add_node("subgraph_node", subgraph)
    .add_edge(START, "subgraph_node")
    .compile(checkpointer=InMemorySaver())
)

# reach into the subgraph's checkpoint
parent_state = graph.get_state(config, subgraphs=True)
sub_config = parent_state.tasks[0].state.config

fork_config = graph.update_state(sub_config, {"value": ["forked"]})
result = graph.invoke(None, fork_config)
# step_b re-runs; step_a's prior result preserved
```

## Debugging Workflow

1. Run the graph normally with a checkpointer.
2. `list(graph.get_state_history(config))` — eyeball `.next` and `.values` per snapshot.
3. Pick a snapshot just before the broken transition and `graph.invoke(None, snap.config)` to replay only that segment.
4. If you need a different starting state, `update_state(snap.config, values=...)` and invoke the returned config.
5. Use `stream_mode="debug"` (see [[LangGraph PT 01 - Streaming]]) on the replay to see every task and checkpoint event.

💡 Replay re-executes — it's not a cache. Use `get_state_history` to find the right `StateSnapshot`, then either invoke its `.config` to retry or `update_state` it to branch.
