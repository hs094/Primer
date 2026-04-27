# DV 01 — Python SDK Overview

🔑 **The Python SDK is a code-first toolkit: define workflows and activities as decorated classes/functions, run them via a worker, drive them with a client.**

## What you build with it

The SDK exposes three primary primitives that map 1:1 to platform concepts:

- **Workflows** — durable, replayable orchestrators. See [[Temporal EN 03 - Workflows]].
- **Activities** — non-deterministic side effects. See [[Temporal EN 04 - Activities]].
- **Workers** — host processes that poll task queues. See [[Temporal EN 06 - Workers]].

Two driver objects:

- **Client** — `temporalio.client.Client` for starting workflows, sending signals/queries/updates.
- **Worker** — `temporalio.worker.Worker` runs registered workflows + activities against a [[Temporal CL 10 - task-queue]].

## Install

```bash
uv add temporalio
```

```python
from temporalio import workflow, activity
from temporalio.client import Client
from temporalio.worker import Worker
```

## Minimal end-to-end shape

```python
@activity.defn
async def say_hello(name: str) -> str:
    return f"Hello, {name}!"

@workflow.defn
class GreetingWorkflow:
    @workflow.run
    async def run(self, name: str) -> str:
        return await workflow.execute_activity(
            say_hello, name,
            start_to_close_timeout=timedelta(seconds=10),
        )

# worker.py
client = await Client.connect("localhost:7233")
worker = Worker(
    client,
    task_queue="hello-tq",
    workflows=[GreetingWorkflow],
    activities=[say_hello],
)
await worker.run()

# starter.py
result = await client.execute_workflow(
    GreetingWorkflow.run, "world",
    id="greeting-1", task_queue="hello-tq",
)
```

## What this guide covers

The Python SDK guide is organised around the same primitives. The notes in this DV series mirror that:

- Workflows: [[Temporal DV 02 - Workflow Basics]], [[Temporal DV 03 - Child Workflows]], [[Temporal DV 04 - Continue As New]], [[Temporal DV 05 - Workflow Cancellation]], [[Temporal DV 06 - Workflow Timeouts]], [[Temporal DV 07 - Message Passing]], [[Temporal DV 08 - Schedules]], [[Temporal DV 09 - Timers]]
- Activities: [[Temporal DV 10 - Activity Basics]], [[Temporal DV 11 - Activity Execution]], [[Temporal DV 12 - Standalone Activities]], [[Temporal DV 13 - Activity Timeouts]], [[Temporal DV 14 - Async Activity Completion]]
- Runtime: [[Temporal DV 15 - Worker Processes]], [[Temporal DV 16 - Temporal Client]], [[Temporal DV 17 - Nexus]]
- Cross-cutting: [[Temporal DV 18 - Observability]], [[Temporal DV 19 - Testing]], [[Temporal DV 20 - Sandbox and Sync vs Async]], [[Temporal DV 21 - Data Handling]], [[Temporal DV 22 - Debugging]]

## Async by default

Every workflow `run`, every activity `defn`, every client call is `async def`. The SDK rides on `asyncio` — workflow code executes inside a deterministic event loop, activity code is plain asyncio.

⚠️ **Workflow code must be deterministic.** No `time.time()`, no `random.random()`, no `requests.get(...)`, no `print` for ops logging. Use `workflow.now()`, `workflow.random()`, `workflow.uuid4()`, `workflow.logger.info(...)`. See [[Temporal DV 20 - Sandbox and Sync vs Async]].

## External resources

- Python API docs (`python.temporal.io`)
- `temporalio/samples-python` for runnable code
- [[Temporal CL 12 - workflow]] and [[Temporal CL 11 - worker]] CLI commands for ops
- Free Temporal 101 course

💡 **Takeaway:** Two decorators (`@workflow.defn`, `@activity.defn`), one async runtime, one client — everything else in this DV series is a refinement on that surface.
