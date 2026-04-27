# DV 02 — Workflow Basics

🔑 **A Workflow is a class decorated with `@workflow.defn`; its entry point is a single `async def` decorated with `@workflow.run`.**

## Define a Workflow

```python
from datetime import timedelta
from temporalio import workflow

# Activities are imported through the sandbox
with workflow.unsafe.imports_passed_through():
    from your_activities import your_activity, YourParams

@workflow.defn(name="YourWorkflow")
class YourWorkflow:
    @workflow.run
    async def run(self, name: str) -> str:
        return await workflow.execute_activity(
            your_activity,
            YourParams("Hello", name),
            start_to_close_timeout=timedelta(seconds=10),
        )
```

The class name is the default Workflow Type; override with `name="..."`. Exactly one method may carry `@workflow.run`. See [[Temporal EN 03 - Workflows]].

## Parameters: prefer a dataclass

Workflows can take any number of args, but the docs strongly recommend a single dataclass for forward compatibility (you can add fields without breaking old callers).

```python
from dataclasses import dataclass

@dataclass
class YourParams:
    greeting: str
    name: str
```

The same applies to return values. Both must be serialisable by the configured payload converter — see [[Temporal DV 21 - Data Handling]].

## Execute an Activity

```python
result = await workflow.execute_activity(
    your_activity,
    YourParams("Hello", name),
    start_to_close_timeout=timedelta(seconds=10),
)
```

`execute_activity` is the helper for `start_activity` + `await handle`. Activity timeouts are *required* — at minimum `start_to_close_timeout` or `schedule_to_close_timeout`. See [[Temporal DV 13 - Activity Timeouts]].

## Register with a Worker

```python
from temporalio.client import Client
from temporalio.worker import Worker

client = await Client.connect("localhost:7233")
worker = Worker(
    client,
    task_queue="your-task-queue",
    workflows=[YourWorkflow],
    activities=[your_activity],
)
await worker.run()
```

The Worker polls the [[Temporal CL 10 - task-queue]] and dispatches Workflow Tasks and Activity Tasks. See [[Temporal DV 15 - Worker Processes]].

## Start a Workflow from a client

```python
# Wait for completion
result = await client.execute_workflow(
    YourWorkflow.run,
    "World",
    id="your-workflow-id",
    task_queue="your-task-queue",
)

# Or fire-and-track via a handle
handle = await client.start_workflow(
    YourWorkflow.run, "World",
    id="your-workflow-id", task_queue="your-task-queue",
)
result = await handle.result()
```

The Workflow ID is the business handle — make it deterministic and meaningful (e.g. `"order-{order_id}"`). See [[Temporal DV 16 - Temporal Client]].

## Determinism rules inside a Workflow

Workflows are **replayed** from [[Temporal EN 07 - Event History]] on every Worker recovery, so the function body must be a pure function of inputs + recorded events.

| Don't | Do |
|---|---|
| `print(...)` | `workflow.logger.info(...)` |
| `random.random()` | `workflow.random().random()` |
| `uuid.uuid4()` | `workflow.uuid4()` |
| `datetime.now()` | `workflow.now()` |
| `time.sleep(5)` | `await asyncio.sleep(5)` (workflow time) |
| `requests.get(...)` | call an Activity |
| threads, global mutation | confine state to `self`, run side effects in activities |

⚠️ **Determinism violations bite at replay, not first execution.** A workflow that uses `datetime.now()` will work fine until a worker restart — then replay diverges and the workflow fails. Lean on [[Temporal DV 20 - Sandbox and Sync vs Async]].

## Logging

```python
workflow.logger.info(f"received name={name}")
```

The workflow logger automatically suppresses messages emitted during replay — safe to use freely.

💡 **Takeaway:** `@workflow.defn` + `@workflow.run` + `workflow.execute_activity` is the entire core API. Treat the workflow body as pure orchestration — anything non-deterministic goes in an activity.
