# FA 01 — Functional API Overview

The Functional API expresses the same Pregel workflow as the [[LangGraph GA 01 - Graph API Overview]], but as plain Python: an `@entrypoint` function that calls `@task`-decorated units of work.
🔑 **Key insight:** `@entrypoint` gives you checkpointing, interrupts, and streaming for control flow written with normal `if`/`for`/function calls — no graph object to define.

## The two decorators

### `@entrypoint`

Marks the workflow's start. The decorated function gets persistence, interrupt support, and a Pregel-backed `.invoke` / `.stream` interface.

```python
from langgraph.func import entrypoint
from langgraph.checkpoint.memory import InMemorySaver

@entrypoint(checkpointer=InMemorySaver())
def my_workflow(some_input: dict) -> int:
    # workflow logic
    return result
```

Async works too:

```python
@entrypoint(checkpointer=checkpointer)
async def my_workflow(some_input: dict) -> int:
    return result
```

**Constraint:** the function must take a single positional argument. Pack multiple values into a dict.

### `@task`

A `@task` is a discrete unit of work — typically an LLM/API call or expensive computation. Calling a task returns a future immediately; results materialize through `.result()` or `await`.

```python
from langgraph.func import task

@task()
def slow_computation(input_value):
    return result
```

**Constraint:** task outputs must be JSON-serializable for checkpointing.

## Futures, sync and async

```python
@entrypoint(checkpointer=checkpointer)
def my_workflow(some_input: int) -> int:
    future = slow_computation(some_input)
    return future.result()
```

Async:

```python
@entrypoint(checkpointer=checkpointer)
async def my_workflow(some_input: int) -> int:
    return await slow_computation(some_input)
```

## Running an entrypoint

The decorated function is itself a `Pregel` object — same surface as a compiled `StateGraph`:

```python
config = {"configurable": {"thread_id": "some_thread_id"}}

my_workflow.invoke(some_input, config)
await my_workflow.ainvoke(some_input, config)

for chunk in my_workflow.stream(some_input, config):
    print(chunk)

async for chunk in my_workflow.astream(some_input, config):
    print(chunk)
```

## Resuming after `interrupt()`

Pause inside the entrypoint with `interrupt(value)`; resume with `Command(resume=...)`:

```python
from langgraph.types import Command, interrupt

my_workflow.invoke(Command(resume=some_resume_value), config)
```

If the run failed for some other reason and you've fixed the cause, resume with `None`:

```python
my_workflow.invoke(None, config)
```

## Injectable parameters

Entrypoints can request extras via keyword-only parameters:

```python
from typing import Any
from langchain_core.runnables import RunnableConfig
from langgraph.store.base import BaseStore
from langgraph.types import StreamWriter

@entrypoint(checkpointer=checkpointer, store=in_memory_store)
def my_workflow(
    some_input: dict,
    *,
    previous: Any = None,    # last invocation's saved value
    store: BaseStore,        # long-term memory
    writer: StreamWriter,    # custom stream output
    config: RunnableConfig,  # runtime config
) -> ...:
    ...
```

## Short-term memory: `previous`

When a checkpointer is configured, `previous` holds the value the entrypoint returned (or `entrypoint.final.save`'d) on the prior invocation of the same thread:

```python
@entrypoint(checkpointer=checkpointer)
def my_workflow(number: int, *, previous: Any = None) -> int:
    previous = previous or 0
    return number + previous

config = {"configurable": {"thread_id": "some_thread_id"}}
my_workflow.invoke(1, config)  # -> 1
my_workflow.invoke(2, config)  # -> 3
```

## Decoupling return from saved state — `entrypoint.final`

Sometimes you want the caller to see one value but persist a different one. `entrypoint.final[ReturnType, SaveType]`:

```python
@entrypoint(checkpointer=checkpointer)
def my_workflow(number: int, *, previous: Any = None) -> entrypoint.final[int, int]:
    previous = previous or 0
    return entrypoint.final(value=previous, save=2 * number)

config = {"configurable": {"thread_id": "1"}}
my_workflow.invoke(3, config)  # returns 0,  saves 6
my_workflow.invoke(1, config)  # returns 6,  saves 2
```

## Functional vs Graph

| Aspect | Functional API | Graph API |
|---|---|---|
| Control flow | Plain Python (`if`, `for`, function calls) | Explicit DAG with edges |
| State | Function-scoped; not shared | Declared `State` + reducers |
| Checkpointing | Saves task results into the existing checkpoint | New checkpoint per super-step |
| Visualization | Not available (dynamic) | Full graph diagrams |

Pick functional when control flow is naturally imperative and you don't need to visualize the topology. Pick the graph API when you want explicit branching, parallel fan-out via `Send`, or a renderable diagram.

## Determinism and side effects

Because `@entrypoint` may *replay* the function body when resuming from a checkpoint, anything non-deterministic (time, randomness, external calls) belongs **inside a task** — task results are checkpointed and won't re-execute on resume.

Wrong — side effect runs twice on resume:

```python
@entrypoint(checkpointer=checkpointer)
def my_workflow(inputs: dict) -> int:
    with open("output.txt", "w") as f:
        f.write("Side effect")  # re-runs after interrupt!
    value = interrupt("question")
    return value
```

Right — side effect inside a task:

```python
@task
def write_to_file():
    with open("output.txt", "w") as f:
        f.write("Side effect")

@entrypoint(checkpointer=checkpointer)
def my_workflow(inputs: dict) -> int:
    write_to_file().result()
    value = interrupt("question")
    return value
```

Treat tasks as idempotent; use idempotency keys on external writes when you can.

## Worked example: human-in-the-loop

```python
import time
from langchain_core.utils.uuid import uuid7
from langgraph.func import entrypoint, task
from langgraph.types import interrupt, Command
from langgraph.checkpoint.memory import InMemorySaver

@task
def write_essay(topic: str) -> str:
    time.sleep(1)
    return f"An essay about topic: {topic}"

@entrypoint(checkpointer=InMemorySaver())
def workflow(topic: str) -> dict:
    essay = write_essay("cat").result()
    is_approved = interrupt({
        "essay": essay,
        "action": "Please approve/reject the essay",
    })
    return {"essay": essay, "is_approved": is_approved}

thread_id = str(uuid7())
config = {"configurable": {"thread_id": thread_id}}

for item in workflow.stream("cat", config):
    print(item)

for item in workflow.stream(Command(resume=True), config):
    print(item)
```

## Serialization rule

Entrypoint inputs and outputs, and all task outputs, must be JSON-serializable when checkpointing is on. Stick to dicts, lists, strings, numbers, booleans, and `None`.

💡 **Takeaway:** `@entrypoint` + `@task` is the smallest API that still gives you persistence and interrupts. When you need explicit edges or `Send` map-reduce, switch to [[LangGraph GA 01 - Graph API Overview]] — both compile to the same Pregel runtime, see [[LangGraph CO 03 - Pregel Runtime]].
