# GA 01 — Graph API Overview

The Graph API models agent workflows as an explicit DAG of **State + Nodes + Edges**, executed by the Pregel runtime in discrete super-steps.
🔑 **Key insight:** nodes do the work, edges decide what runs next. State is shared, and reducers control how each node's update is merged into it.

## The three primitives

- **State** — a typed shared structure (`TypedDict`, `dataclass`, or Pydantic `BaseModel`) passed to every node.
- **Nodes** — Python callables that take state (and optionally `config` / `runtime`) and return a partial update.
- **Edges** — wiring (static, conditional, or `Send`-based) that decides what runs next.

See [[LangGraph CO 03 - Pregel Runtime]] for how these execute as super-steps.

## Defining state

```python
from typing_extensions import TypedDict

class State(TypedDict):
    foo: int
    bar: list[str]
```

`dataclass` works too (and supports defaults):

```python
from dataclasses import dataclass

@dataclass
class State:
    foo: int = 0
    bar: list[str] = None
```

Pydantic `BaseModel` is supported but slower than `TypedDict`:

```python
from pydantic import BaseModel

class State(BaseModel):
    foo: int
    bar: list[str]
```

## Reducers

Default behavior **overwrites** a key. To merge instead, annotate the field with a reducer.

```python
from typing import Annotated
from operator import add
from typing_extensions import TypedDict

class State(TypedDict):
    foo: int
    bar: Annotated[list[str], add]
```

For chat history, use `add_messages` — it appends, dedupes by id, and supports `RemoveMessage`:

```python
from typing import Annotated
from typing_extensions import TypedDict
from langgraph.graph.message import add_messages
from langchain.messages import AnyMessage

class GraphState(TypedDict):
    messages: Annotated[list[AnyMessage], add_messages]
```

`MessagesState` is the prebuilt shorthand:

```python
from langgraph.graph import MessagesState

class State(MessagesState):
    documents: list[str]
```

## Nodes

A node is just a function. Return a dict containing the keys it wants to update.

```python
def plain_node(state: State):
    return state
```

Optional injected params: `config: RunnableConfig`, `runtime: Runtime[ContextSchema]`.

```python
from dataclasses import dataclass
from langgraph.runtime import Runtime

@dataclass
class Context:
    user_id: str

def node_with_runtime(state: State, runtime: Runtime[Context]):
    print("User ID:", runtime.context.user_id)
    return {"results": f"Hello, {state['input']}!"}
```

Add nodes by name, or implicitly using the function name:

```python
from langgraph.graph import StateGraph

builder = StateGraph(State)
builder.add_node("node_name", node_function)
builder.add_node(plain_node)  # name = "plain_node"
```

## Edges

Static edges, plus the special `START` and `END` sentinels:

```python
from langgraph.graph import START, END

graph.add_edge(START, "node_a")
graph.add_edge("node_a", "node_b")
graph.add_edge("node_b", END)
```

Conditional edges branch on a routing function's return value:

```python
def routing_function(state: State):
    if state["foo"] == "bar":
        return "node_b"
    return "node_c"

graph.add_conditional_edges("node_a", routing_function)
```

With an explicit mapping (router returns a key, mapping picks the node):

```python
graph.add_conditional_edges(
    "node_a",
    routing_function,
    {True: "node_b", False: "node_c"},
)
```

`START` itself can be a conditional source, giving you a conditional entry point.

## Compile and run

```python
graph = builder.compile()
result = graph.invoke({"input": "data"})

for chunk in graph.stream({"input": "data"}):
    print(chunk)
```

`compile()` is required before invocation. Pass a `checkpointer=` to enable persistence; see [[LangGraph CO 05 - Persistence (Checkpointers)]].

## Runtime context

Static-per-run config (model provider, user id, feature flags) goes through `context_schema`, not state:

```python
from dataclasses import dataclass
from langgraph.graph import StateGraph

@dataclass
class ContextSchema:
    llm_provider: str = "openai"

builder = StateGraph(State, context_schema=ContextSchema)
```

Pass it in at invocation, read it via `Runtime`:

```python
graph.invoke(inputs, context={"llm_provider": "anthropic"})
```

## Recursion guard

The runtime halts after `recursion_limit` super-steps (default 25). To wind down gracefully instead of erroring, watch `RemainingSteps`:

```python
from langgraph.managed import RemainingSteps

class State(TypedDict):
    messages: Annotated[list, lambda x, y: x + y]
    remaining_steps: RemainingSteps
```

## Mental model

- One super-step = one parallel batch of node executions.
- Each node returns a partial state update; reducers merge it.
- The graph halts when no node has anything to run and no `Send` is in flight.

💡 **Takeaway:** pick `StateGraph` when control flow is structured and inspectable. Reach for the [[LangGraph FA 01 - Functional API Overview]] when you'd rather express the same workflow as plain Python with `@entrypoint` and `@task`.
