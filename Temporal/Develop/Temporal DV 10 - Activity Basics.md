# DV 10 — Activity Basics

🔑 Activities are functions for the **side-effecty** parts of your application — calling APIs, hitting databases, doing CPU work. They are the only place a Workflow is allowed to be non-deterministic. In Python, you mark a function with `@activity.defn` and either `async def` or plain `def`.

## Defining an Activity

```python
from temporalio import activity


@activity.defn
async def your_activity(input: YourParams) -> str:
    return f"{input.greeting}, {input.name}!"
```

The function becomes an Activity Definition the Worker can pick up off a Task Queue. See [[Temporal EN 04 - Activities]].

## Custom Activity Name

By default the Activity's registered name is the function name. Override it with `name=`:

```python
@activity.defn(name="your_activity")
async def your_activity(input: YourParams) -> str:
    return f"{input.greeting}, {input.name}!"
```

This is what Workflows reference when calling [[Temporal DV 11 - Activity Execution]] via string-name dispatch.

## Three Execution Models

The SDK supports three implementation styles. Pick based on what your code blocks on:

| Style | Definition | Use For |
|---|---|---|
| **Async** | `async def`, `asyncio` | I/O-bound (HTTP, DB) |
| **Sync threaded** | `def` + `concurrent.futures.ThreadPoolExecutor` | Blocking libs without async equivalents |
| **Sync multiprocess** | `def` + `concurrent.futures.ProcessPoolExecutor` | CPU-heavy work |

⚠️ **Never block the asyncio event loop inside an `async def` Activity.** Doing so "converts asynchronous execution into serial processing" and crushes Worker throughput. Either go fully async, or define the Activity as plain `def` and run it on a thread/process executor.

## Activity Context

Inside an Activity, `activity.info()` exposes runtime metadata:

```python
@activity.defn
async def my_activity(name: str) -> str:
    info = activity.info()
    activity.logger.info(f"attempt {info.attempt} of {info.activity_type}")
    return f"hello {name}"
```

`activity.logger` is a structured logger pre-attached to the Activity context. Heartbeats also live on this module — see [[Temporal DV 13 - Activity Timeouts]].

## Parameters and Return Values

> "Use a single object as an argument that wraps the application data passed to Activities."

Why: adding a field to a dataclass is backwards-compatible, but adding a positional parameter is not. Wrap inputs in a Pydantic model or dataclass.

```python
from dataclasses import dataclass


@dataclass
class YourParams:
    greeting: str
    name: str


@activity.defn
async def your_activity(input: YourParams) -> str:
    return f"{input.greeting}, {input.name}!"
```

⚠️ **Payload size limits**: individual arguments max **2 MB**, total gRPC messages max **4 MB**. Anything larger should live in object storage with the Activity passing a reference (S3 key, blob URL).

⚠️ **All return values must be serializable** by the configured data converter. Default JSON converter handles dataclasses, Pydantic models, and primitives.

## Where to Put Activities

Convention: keep Activities in their own module, separate from Workflows.

```
your_project/
├── activities.py     # @activity.defn functions
├── workflows.py      # @workflow.defn classes
├── shared.py         # dataclasses / Pydantic models
└── worker.py         # Worker(activities=[...], workflows=[...])
```

Workflows import the Activity **function reference** for type-safe calls; the runtime never imports Workflow code into the Activity sandbox.

## See Also

- [[Temporal DV 11 - Activity Execution]] — how Workflows call Activities.
- [[Temporal DV 12 - Standalone Activities]] — running Activities without a Workflow.
- [[Temporal DV 13 - Activity Timeouts]] — `start_to_close_timeout` and friends.
- [[Temporal CL 03 - activity]] — CLI ops on Activities.

💡 **Takeaway:** `@activity.defn` turns a function into a Temporal Activity. Pass a single dataclass argument, pick async/threaded/multiprocess to match your I/O profile, and never block the asyncio loop in an `async def` Activity.
