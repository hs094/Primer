# FA 02 — Using the Functional API

Recipes for the things you actually do with `@entrypoint` and `@task`: multi-arg input, parallel fan-out, custom streaming, interrupts, persistence, and mixing in graphs.
🔑 **Key insight:** the orchestration is plain Python — list comprehensions for fan-out, `.result()` for joins, `interrupt()` for human-in-the-loop. The decorators just give it durability.

## Multiple inputs

`@entrypoint` takes a single positional argument. Pack extras into a dict:

```python
@entrypoint(checkpointer=checkpointer)
def my_workflow(inputs: dict) -> int:
    value = inputs["value"]
    another_value = inputs["another_value"]
    ...

my_workflow.invoke({"value": 1, "another_value": 2})
```

## Parallel tasks

Kick off tasks together, then resolve futures with `.result()`:

```python
@task
def add_one(number: int) -> int:
    return number + 1

@entrypoint(checkpointer=checkpointer)
def graph(numbers: list[int]) -> list[int]:
    futures = [add_one(i) for i in numbers]
    return [f.result() for f in futures]
```

This is the functional equivalent of fan-out / fan-in in [[LangGraph GA 02 - Using the Graph API]]. The runtime schedules the tasks concurrently; `.result()` blocks until each completes.

## Custom streaming

Use `get_stream_writer` (or the injected `writer: StreamWriter`) to push custom messages into the stream alongside `updates`:

```python
from langgraph.config import get_stream_writer
from langgraph.checkpoint.memory import InMemorySaver

checkpointer = InMemorySaver()

@entrypoint(checkpointer=checkpointer)
def main(inputs: dict) -> int:
    writer = get_stream_writer()
    writer("Started processing")
    result = inputs["x"] * 2
    writer(f"Result is {result}")
    return result

config = {"configurable": {"thread_id": "abc"}}

for mode, chunk in main.stream(
    {"x": 5},
    stream_mode=["custom", "updates"],
    config=config,
):
    print(f"{mode}: {chunk}")
```

`stream_mode=["custom", "updates"]` interleaves your `writer(...)` payloads (mode `custom`) with normal task updates.

## Human-in-the-loop with `interrupt()`

`interrupt(value)` halts the workflow and surfaces `value` to the caller. `Command(resume=...)` resumes:

```python
from langgraph.func import entrypoint, task
from langgraph.types import Command, interrupt
from langgraph.checkpoint.memory import InMemorySaver

@task
def step_1(input_query):
    return f"{input_query} bar"

@task
def human_feedback(input_query):
    feedback = interrupt(f"Please provide feedback: {input_query}")
    return f"{input_query} {feedback}"

@task
def step_3(input_query):
    return f"{input_query} qux"

checkpointer = InMemorySaver()

@entrypoint(checkpointer=checkpointer)
def graph(input_query):
    result_1 = step_1(input_query).result()
    result_2 = human_feedback(result_1).result()
    result_3 = step_3(result_2).result()
    return result_3
```

Drive it:

```python
config = {"configurable": {"thread_id": "1"}}

for event in graph.stream("foo", config):
    print(event)

for event in graph.stream(Command(resume="baz"), config):
    print(event)
```

Note that `interrupt()` is called from within a `@task` so that the surrounding logic doesn't re-execute on resume — see the side-effect rule in [[LangGraph FA 01 - Functional API Overview]].

## Checkpointing and persistence

Any `BaseCheckpointSaver` works. `InMemorySaver` is fine for local dev:

```python
from langgraph.checkpoint.memory import InMemorySaver
from langchain_core.utils.uuid import uuid7

checkpointer = InMemorySaver()

@entrypoint(checkpointer=checkpointer)
def workflow(inputs: dict) -> str:
    even = is_even(inputs["number"]).result()
    return format_message(even).result()

config = {"configurable": {"thread_id": str(uuid7())}}
result = workflow.invoke({"number": 7}, config=config)
```

For SQLite/Postgres-backed savers see [[LangGraph CO 05 - Persistence (Checkpointers)]].

## Inspecting thread state

The Pregel object exposes `get_state` and `get_state_history`:

```python
config = {"configurable": {"thread_id": "1"}}

graph.get_state(config)               # latest checkpoint
list(graph.get_state_history(config)) # all checkpoints, newest first
```

## Decouple return value from saved value — `entrypoint.final`

Useful when the caller wants the *previous* value but the next invocation should start from something different:

```python
@entrypoint(checkpointer=checkpointer)
def accumulate(n: int, *, previous: int | None) -> entrypoint.final[int, int]:
    previous = previous or 0
    total = previous + n
    return entrypoint.final(value=previous, save=total)

config = {"configurable": {"thread_id": "my-thread"}}

print(accumulate.invoke(1, config=config))  # returns 0, saves 1
print(accumulate.invoke(2, config=config))  # returns 1, saves 3
```

## Calling Graph-API graphs from an entrypoint

Mix freely — a compiled graph is just another callable:

```python
from langgraph.func import entrypoint
from langgraph.graph import StateGraph

builder = StateGraph(...)
some_graph = builder.compile()

@entrypoint()
def some_workflow(some_input: dict) -> dict:
    result_1 = some_graph.invoke(...)
    result_2 = another_graph.invoke(...)
    return {"result_1": result_1, "result_2": result_2}
```

This is the standard escape hatch when one branch of your workflow naturally wants the explicit graph API (e.g., `Send` map-reduce) but the rest is fine as imperative Python.

## Task-level retry

Configure retries per task with `RetryPolicy`:

```python
from langgraph.types import RetryPolicy

retry_policy = RetryPolicy(retry_on=ValueError)

@task(retry_policy=retry_policy)
def get_info():
    ...
```

## Task-level caching

Memoize a task with a TTL — useful for expensive idempotent calls inside the same workflow:

```python
import time
from langgraph.cache.memory import InMemoryCache
from langgraph.types import CachePolicy

@task(cache_policy=CachePolicy(ttl=120))
def slow_add(x: int) -> int:
    time.sleep(1)
    return x * 2

@entrypoint(cache=InMemoryCache())
def main(inputs: dict) -> dict[str, int]:
    result1 = slow_add(inputs["x"]).result()
    result2 = slow_add(inputs["x"]).result()  # served from cache
    return {"result1": result1, "result2": result2}
```

## When to use which mode

| Need | Reach for |
|---|---|
| Run many tasks concurrently | List of futures + `.result()` |
| Push intermediate UI updates | `get_stream_writer()` + `stream_mode=["custom","updates"]` |
| Pause for a human | `interrupt()` inside a `@task`, resume with `Command(resume=...)` |
| Carry value across calls on a thread | `previous` parameter, or `entrypoint.final(save=...)` |
| Cap external-call cost | `@task(cache_policy=CachePolicy(ttl=...))` + `cache=InMemoryCache()` on entrypoint |
| Branch into an explicit DAG | Call a compiled `StateGraph` from inside the entrypoint |

💡 **Takeaway:** keep all non-deterministic work and external side effects inside `@task`s, let the `@entrypoint` body be the orchestration. That keeps resume-after-interrupt safe and makes everything you wrote about [[LangGraph FA 01 - Functional API Overview]] hold true.
