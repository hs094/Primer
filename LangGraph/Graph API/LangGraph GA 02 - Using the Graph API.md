# GA 02 — Using the Graph API

Recipes for the things you actually do with `StateGraph`: schemas, message reducers, runtime context, retries, parallel branches, loops, and `Command`-driven routing.
🔑 **Key insight:** the API is small — most "features" are just disciplined uses of state, reducers, and edges. Pick the right primitive and the rest follows.

## Input / output schemas

Decouple the public input from the public output by passing both to `StateGraph`:

```python
from langgraph.graph import StateGraph, START, END
from typing_extensions import TypedDict

class InputState(TypedDict):
    question: str

class OutputState(TypedDict):
    answer: str

class OverallState(InputState, OutputState):
    pass

def answer_node(state: InputState):
    return {"answer": "bye", "question": state["question"]}

builder = StateGraph(OverallState, input_schema=InputState, output_schema=OutputState)
builder.add_node(answer_node)
builder.add_edge(START, "answer_node")
builder.add_edge("answer_node", END)
graph = builder.compile()

print(graph.invoke({"question": "hi"}))
# {'answer': 'bye'}
```

## Private channels between nodes

Use a node-local schema as a typed "channel" two nodes share without exposing it on the public state:

```python
from langgraph.graph import StateGraph, START
from typing_extensions import TypedDict

class OverallState(TypedDict):
    a: str

class Node1Output(TypedDict):
    private_data: str

def node_1(state: OverallState) -> Node1Output:
    return {"private_data": "set by node_1"}

class Node2Input(TypedDict):
    private_data: str

def node_2(state: Node2Input) -> OverallState:
    return {"a": "set by node_2"}

def node_3(state: OverallState) -> OverallState:
    return {"a": "set by node_3"}

builder = StateGraph(OverallState).add_sequence([node_1, node_2, node_3])
builder.add_edge(START, "node_1")
graph = builder.compile()
```

## Message reducers

`add_messages` appends, deduplicates by id, and respects `RemoveMessage`:

```python
from langgraph.graph.message import add_messages
from typing_extensions import Annotated, TypedDict
from langchain.messages import AnyMessage, AIMessage

class State(TypedDict):
    messages: Annotated[list[AnyMessage], add_messages]
    extra_field: int

def node(state: State):
    return {"messages": [AIMessage("Hello!")], "extra_field": 10}
```

`MessagesState` is the same thing, ready-made:

```python
from langgraph.graph import MessagesState

class State(MessagesState):
    extra_field: int
```

To bypass a reducer for one update (full replace), wrap with `Overwrite`:

```python
from langgraph.types import Overwrite
import operator

class State(TypedDict):
    messages: Annotated[list, operator.add]

def replace_messages(state: State):
    return {"messages": Overwrite(["replacement message"])}
```

JSON-friendly form: `{"messages": {"__overwrite__": [...]}}`.

## Runtime context

Pass per-invocation static config via `context=` and read it through `Runtime`:

```python
from langgraph.graph import StateGraph, START, END
from langgraph.runtime import Runtime
from typing_extensions import TypedDict

class ContextSchema(TypedDict):
    my_runtime_value: str

class State(TypedDict):
    my_state_value: str

def node(state: State, runtime: Runtime[ContextSchema]):
    if runtime.context["my_runtime_value"] == "a":
        return {"my_state_value": 1}
    return {"my_state_value": 2}

builder = StateGraph(State, context_schema=ContextSchema)
builder.add_node(node)
builder.add_edge(START, "node")
builder.add_edge("node", END)
graph = builder.compile()

graph.invoke({}, context={"my_runtime_value": "a"})
```

Common pattern — pick the LLM at runtime:

```python
from dataclasses import dataclass
from langchain.chat_models import init_chat_model
from langgraph.graph import MessagesState, StateGraph, START, END
from langgraph.runtime import Runtime

@dataclass
class ContextSchema:
    model_provider: str = "anthropic"

MODELS = {
    "anthropic": init_chat_model("claude-haiku-4-5-20251001"),
    "openai":    init_chat_model("gpt-5.4-mini"),
}

def call_model(state: MessagesState, runtime: Runtime[ContextSchema]):
    model = MODELS[runtime.context.model_provider]
    return {"messages": [model.invoke(state["messages"])]}

builder = StateGraph(MessagesState, context_schema=ContextSchema)
builder.add_node("model", call_model)
builder.add_edge(START, "model")
builder.add_edge("model", END)
graph = builder.compile()
```

## Retries

Per-node `RetryPolicy` — retry on a specific exception or with a max-attempts cap:

```python
import sqlite3
from langgraph.types import RetryPolicy

builder.add_node(
    "query_database",
    query_database,
    retry_policy=RetryPolicy(retry_on=sqlite3.OperationalError),
)
builder.add_node("model", call_model, retry_policy=RetryPolicy(max_attempts=5))
```

Inside the node, branch on retry attempt via `runtime.execution_info`:

```python
def my_node(state: State, runtime: Runtime):
    info = runtime.execution_info
    if info.node_attempt > 1:
        return {"result": "fallback_result"}
    return {"result": "primary_result"}
```

## Node caching

```python
from langgraph.cache.memory import InMemoryCache
from langgraph.types import CachePolicy

builder.add_node("expensive_node", expensive_node, cache_policy=CachePolicy(ttl=3))
graph = builder.compile(cache=InMemoryCache())
```

## Sequences

```python
builder = StateGraph(State).add_sequence([step_1, step_2, step_3])
builder.add_edge(START, "step_1")
graph = builder.compile()
```

## Parallel fan-out / fan-in

Two outgoing edges from `a` run in the same super-step; reducers merge results:

```python
import operator
from typing import Annotated
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END

class State(TypedDict):
    aggregate: Annotated[list, operator.add]

def a(state): return {"aggregate": ["A"]}
def b(state): return {"aggregate": ["B"]}
def c(state): return {"aggregate": ["C"]}
def d(state): return {"aggregate": ["D"]}

builder = StateGraph(State)
for n in (a, b, c, d): builder.add_node(n)
builder.add_edge(START, "a")
builder.add_edge("a", "b")
builder.add_edge("a", "c")
builder.add_edge("b", "d")
builder.add_edge("c", "d")
builder.add_edge("d", END)
graph = builder.compile()
```

`defer=True` postpones a node until all paths leading to it finish, useful for funnels:

```python
builder.add_node(d, defer=True)
```

Cap parallelism with `max_concurrency`:

```python
graph.invoke(inputs, {"configurable": {"max_concurrency": 10}})
```

## Map-reduce with `Send`

`Send` dispatches a node multiple times, each with a *different* state slice:

```python
from langgraph.types import Send
from langgraph.graph import StateGraph, START, END
from typing_extensions import TypedDict, Annotated
import operator

class OverallState(TypedDict):
    topic: str
    subjects: list[str]
    jokes: Annotated[list[str], operator.add]
    best_selected_joke: str

def generate_topics(state):
    return {"subjects": ["lions", "elephants", "penguins"]}

def generate_joke(state):
    return {"jokes": [f"joke about {state['subject']}"]}

def continue_to_jokes(state):
    return [Send("generate_joke", {"subject": s}) for s in state["subjects"]]

def best_joke(state):
    return {"best_selected_joke": "penguins"}

builder = StateGraph(OverallState)
builder.add_node("generate_topics", generate_topics)
builder.add_node("generate_joke", generate_joke)
builder.add_node("best_joke", best_joke)
builder.add_edge(START, "generate_topics")
builder.add_conditional_edges("generate_topics", continue_to_jokes, ["generate_joke"])
builder.add_edge("generate_joke", "best_joke")
builder.add_edge("best_joke", END)
graph = builder.compile()
```

## `Command`: update + route from a node

`Command` lets a node both write state and choose its successor — no separate routing function needed:

```python
import random
from typing_extensions import TypedDict, Literal
from langgraph.graph import StateGraph, START
from langgraph.types import Command

class State(TypedDict):
    foo: str

def node_a(state) -> Command[Literal["node_b", "node_c"]]:
    goto = "node_b" if random.choice(["b", "c"]) == "b" else "node_c"
    return Command(update={"foo": goto[-1]}, goto=goto)
```

To target a node in the parent graph from a subgraph, set `graph=Command.PARENT`:

```python
return Command(update={"foo": "b"}, goto="node_b", graph=Command.PARENT)
```

## Loops & recursion limits

Conditional edge as the loop guard:

```python
def route(state) -> Literal["b", END]:
    return END if len(state["aggregate"]) >= 7 else "b"

builder.add_conditional_edges("a", route)
builder.add_edge("b", "a")
```

Catch the limit:

```python
from langgraph.errors import GraphRecursionError

try:
    graph.invoke({"aggregate": []}, {"recursion_limit": 4})
except GraphRecursionError:
    print("Recursion Error")
```

Wind down gracefully via `RemainingSteps`:

```python
from langgraph.managed.is_last_step import RemainingSteps

class State(TypedDict):
    aggregate: Annotated[list, operator.add]
    remaining_steps: RemainingSteps

def route(state) -> Literal["b", END]:
    return END if state["remaining_steps"] <= 2 else "b"
```

## Async

Just declare `async def` nodes and call `ainvoke` / `astream`:

```python
async def node(state: MessagesState):
    llm = init_chat_model("claude-haiku-4-5-20251001")
    return {"messages": [await llm.ainvoke(state["messages"])]}

result = await graph.ainvoke({"messages": [{"role": "user", "content": "Hello"}]})
```

💡 **Takeaway:** reach for `Command` when routing depends on the node's output, `Send` for map-reduce, `RetryPolicy` + `RemainingSteps` for resilience, and `defer=True` for funnels. See [[LangGraph GA 03 - Subgraphs]] for composing graphs.
