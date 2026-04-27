# GA 03 — Subgraphs

A subgraph is a compiled `StateGraph` used as a node inside another graph. The choice you have to make is whether parent and child share schema or need translation between them.
🔑 **Key insight:** matching schemas → drop the compiled subgraph in as a node; mismatched schemas → wrap the call in a function that maps state in and out.

## Pattern 1 — Shared schema (drop in as a node)

When parent and child share the relevant state keys, `add_node` accepts the compiled subgraph directly:

```python
from typing_extensions import TypedDict
from langgraph.graph.state import StateGraph, START

class State(TypedDict):
    foo: str

def subgraph_node_1(state: State):
    return {"foo": "hi! " + state["foo"]}

subgraph_builder = StateGraph(State)
subgraph_builder.add_node(subgraph_node_1)
subgraph_builder.add_edge(START, "subgraph_node_1")
subgraph = subgraph_builder.compile()

builder = StateGraph(State)
builder.add_node("node_1", subgraph)  # compiled subgraph as a node
builder.add_edge(START, "node_1")
graph = builder.compile()
```

## Pattern 2 — Different schemas (wrap in a function)

Translate at the boundary by writing a normal node that invokes the subgraph:

```python
from typing_extensions import TypedDict
from langgraph.graph.state import StateGraph, START

class SubgraphState(TypedDict):
    bar: str

def subgraph_node_1(state: SubgraphState):
    return {"bar": "hi! " + state["bar"]}

subgraph_builder = StateGraph(SubgraphState)
subgraph_builder.add_node(subgraph_node_1)
subgraph_builder.add_edge(START, "subgraph_node_1")
subgraph = subgraph_builder.compile()

class State(TypedDict):
    foo: str

def call_subgraph(state: State):
    out = subgraph.invoke({"bar": state["foo"]})
    return {"foo": out["bar"]}

builder = StateGraph(State)
builder.add_node("node_1", call_subgraph)
builder.add_edge(START, "node_1")
graph = builder.compile()
```

## Multi-level nesting

Each level can compose the next using either pattern:

```python
from typing_extensions import TypedDict
from langgraph.graph.state import StateGraph, START, END

class GrandChildState(TypedDict):
    my_grandchild_key: str

def grandchild_1(state: GrandChildState) -> GrandChildState:
    return {"my_grandchild_key": state["my_grandchild_key"] + ", how are you"}

grandchild = StateGraph(GrandChildState)
grandchild.add_node("grandchild_1", grandchild_1)
grandchild.add_edge(START, "grandchild_1")
grandchild.add_edge("grandchild_1", END)
grandchild_graph = grandchild.compile()

class ChildState(TypedDict):
    my_child_key: str

def call_grandchild_graph(state: ChildState) -> ChildState:
    out = grandchild_graph.invoke({"my_grandchild_key": state["my_child_key"]})
    return {"my_child_key": out["my_grandchild_key"] + " today?"}

child = StateGraph(ChildState)
child.add_node("child_1", call_grandchild_graph)
child.add_edge(START, "child_1")
child.add_edge("child_1", END)
child_graph = child.compile()

class ParentState(TypedDict):
    my_key: str

def parent_1(state): return {"my_key": "hi " + state["my_key"]}
def parent_2(state): return {"my_key": state["my_key"] + " bye!"}

def call_child_graph(state: ParentState) -> ParentState:
    out = child_graph.invoke({"my_child_key": state["my_key"]})
    return {"my_key": out["my_child_key"]}

parent = StateGraph(ParentState)
parent.add_node("parent_1", parent_1)
parent.add_node("child", call_child_graph)
parent.add_node("parent_2", parent_2)
parent.add_edge(START, "parent_1")
parent.add_edge("parent_1", "child")
parent.add_edge("child", "parent_2")
parent.add_edge("parent_2", END)
parent_graph = parent.compile()
```

## Persistence options

Three flavors. Pick by how much of the subgraph's state you want to keep around.

### Per-invocation (default)

The subgraph runs fresh each call but inherits the parent's checkpointer so interrupts still work:

```python
from langchain.agents import create_agent
from langgraph.checkpoint.memory import MemorySaver

fruit_agent = create_agent(model="gpt-4-mini", tools=[fruit_info], prompt="...")
veggie_agent = create_agent(model="gpt-4-mini", tools=[veggie_info], prompt="...")

agent = create_agent(
    model="gpt-4-mini",
    tools=[ask_fruit_expert, ask_veggie_expert],
    prompt="Delegate to the right expert.",
    checkpointer=MemorySaver(),
)
```

Resume after `interrupt()`:

```python
from langgraph.types import Command

config = {"configurable": {"thread_id": "1"}}
agent.invoke({"messages": [{"role": "user", "content": "Tell me about apples"}]}, config)
agent.invoke(Command(resume=True), config)
```

### Per-thread

Subagent state accumulates across calls on the same thread — pass `checkpointer=True` so it picks up the parent's checkpointer:

```python
fruit_agent = create_agent(
    model="gpt-4-mini",
    tools=[fruit_info],
    prompt="...",
    checkpointer=True,  # share parent's checkpointer per-thread
)
```

When using multiple per-thread subagents, give each its own node name to avoid checkpoint collisions:

```python
from langgraph.graph import MessagesState, StateGraph

def create_sub_agent(model, *, name, **kwargs):
    agent = create_agent(model=model, name=name, **kwargs)
    return (
        StateGraph(MessagesState)
        .add_node(name, agent)
        .add_edge("__start__", name)
        .compile()
    )
```

### Stateless

Disable checkpointing entirely:

```python
subgraph = subgraph_builder.compile(checkpointer=False)
```

See [[LangGraph CO 05 - Persistence (Checkpointers)]] for the persistence model.

## Streaming from subgraphs

Set `subgraphs=True` (and `version="v2"`) to receive subgraph events alongside parent events. Each chunk has `type`, `ns` (namespace tuple identifying which subgraph), and `data`:

```python
for chunk in graph.stream(
    {"foo": "foo"},
    subgraphs=True,
    stream_mode="updates",
    version="v2",
):
    if chunk["type"] == "updates":
        print(chunk["ns"], chunk["data"])
```

Output looks like:

```
() {'node_1': {'foo': 'hi! foo'}}
('node_2:e58e5673-...',) {'subgraph_node_1': {'bar': 'bar'}}
('node_2:e58e5673-...',) {'subgraph_node_2': {'foo': 'hi! foobar'}}
() {'node_2': {'foo': 'hi! foobar'}}
```

`ns == ()` is the root graph; non-empty tuples identify the subgraph instance.

## Inspecting subgraph state

Use `get_state(config, subgraphs=True)` to descend into the child:

```python
from langgraph.checkpoint.memory import MemorySaver
from langgraph.types import interrupt, Command

def subgraph_node_1(state):
    value = interrupt("Provide value:")
    return {"foo": state["foo"] + value}

# ... compile parent with a MemorySaver

config = {"configurable": {"thread_id": "1"}}
graph.invoke({"foo": ""}, config)

subgraph_state = graph.get_state(config, subgraphs=True).tasks[0].state

graph.invoke(Command(resume="bar"), config)
```

## Quick decision table

| Situation | Approach |
|---|---|
| Subgraph shares the relevant state keys | `add_node("name", compiled_subgraph)` |
| Schemas differ | Wrap in a node function that translates |
| Subagents must be independent each call | Per-invocation (default) |
| Subagent needs its own multi-turn memory | Per-thread (`checkpointer=True`), unique node names |
| No persistence at all | `compile(checkpointer=False)` |
| Want to observe subgraph events | `stream(..., subgraphs=True, version="v2")` |

💡 **Takeaway:** subgraphs are graphs all the way down. Match schemas to keep wiring trivial; translate at the boundary when you can't. Reach for `Command(graph=Command.PARENT)` from inside a subgraph when a child needs to jump to a sibling node in the parent — see [[LangGraph GA 02 - Using the Graph API]].
