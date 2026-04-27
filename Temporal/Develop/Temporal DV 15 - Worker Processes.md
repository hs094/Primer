# DV 15 — Worker Processes

🔑 The **Worker Process** is where Workflow and Activity functions actually execute. It long-polls a Task Queue, picks up tasks, runs your code, and reports results back to the Temporal Service. No Worker = nothing happens.

## What a Worker is

A **Worker Entity** contains a Workflow Worker and/or an Activity Worker. Each Worker:

- Registers exact Workflow Types and Activity Types it can execute.
- Associates with **exactly one** Task Queue.
- Long-polls that queue for tasks.

Multiple Worker Entities polling the **same Task Queue** must register the **identical** set of Workflows and Activities — otherwise tasks may land on a Worker that can't handle them.

⚠️ If a Worker receives a task for an unknown Workflow or Activity Type, it fails the **task** (not the Workflow Execution). The Workflow keeps retrying until a capable Worker shows up.

## Minimal Worker

```python
import asyncio
from temporalio.client import Client
from temporalio.worker import Worker

async def main():
    client = await Client.connect("localhost:7233")
    worker = Worker(
        client,
        task_queue="your-task-queue",
        workflows=[YourWorkflow],
        activities=[your_activity],
    )
    await worker.run()

if __name__ == "__main__":
    asyncio.run(main())
```

`worker.run()` blocks forever and pumps tasks until the process is killed or `worker.shutdown()` is called.

## Registering types

| Param | What goes here |
|---|---|
| `workflows=[...]` | Workflow **classes** (decorated with `@workflow.defn`) |
| `activities=[...]` | Activity **functions** (decorated with `@activity.defn`) — async or sync |
| `task_queue=` | String name; must match what the Client uses to start work |

You can pass instance methods too — bind them once and pass the bound method:

```python
acts = TranslateActivities(session)
worker = Worker(
    client,
    task_queue="translate",
    workflows=[GreetingWorkflow],
    activities=[acts.greet_in_spanish, acts.greet_in_french],
)
```

## Running multiple Workers per process

A single process can host multiple Workers on different Task Queues:

```python
async with Worker(client, task_queue="q1", workflows=[A], activities=[a]):
    async with Worker(client, task_queue="q2", workflows=[B], activities=[b]):
        await asyncio.Future()  # run forever
```

The `async with` form gives you graceful shutdown when the block exits.

## Programmatic shutdown

`worker.run()` accepts cancellation. Common pattern:

```python
async def main():
    client = await Client.connect("localhost:7233")
    worker = Worker(client, task_queue="q", workflows=[W], activities=[a])
    task = asyncio.create_task(worker.run())

    # ... later, on signal ...
    await worker.shutdown()
    await task
```

`shutdown()` stops polling immediately, then waits for in-flight tasks to finish or hit their `graceful_shutdown_timeout`.

## Sizing knobs

Pass these to `Worker(...)` to tune throughput. See [[Temporal BP 02 - Worker Performance]] for the full guide.

| Param | Default | Notes |
|---|---|---|
| `max_concurrent_activities` | 100 | Concurrent Activity Tasks |
| `max_concurrent_workflow_tasks` | 100 | Concurrent Workflow Tasks |
| `max_concurrent_local_activities` | 100 | For local activities |
| `activity_executor` | None | Required for **sync** activities — pass a `ThreadPoolExecutor` |

```python
from concurrent.futures import ThreadPoolExecutor

with ThreadPoolExecutor(max_workers=42) as executor:
    worker = Worker(
        client,
        task_queue="q",
        workflows=[W],
        activities=[sync_activity],
        activity_executor=executor,
    )
    await worker.run()
```

⚠️ Sync activities **must** have an `activity_executor`. Async activities run on the asyncio loop and don't need one — see [[Temporal DV 20 - Sandbox and Sync vs Async]].

## Operating Workers

Use the [[Temporal CL 11 - worker]] CLI commands to inspect Workers from the outside:

```bash
temporal task-queue describe --task-queue your-task-queue
```

This lists pollers (Worker identities currently long-polling the queue) — useful to confirm Workers are connected.

## Topology rules of thumb

- **One process per role**: separate Worker processes for separate Task Queues lets you scale each role independently.
- **GIL limits CPU**: a single Python process is single-core for CPU-bound work. Run multiple processes to use more cores. See [[Temporal DV 20 - Sandbox and Sync vs Async]].
- **Workers are stateless** from Temporal's POV — kill them, replace them, scale horizontally. State lives in the Service.

## Cross-references

- [[Temporal EN 06 - Workers]] — Worker concepts in depth
- [[Temporal EN 02 - Temporal SDKs]] — what the SDK gives you
- [[Temporal DV 16 - Temporal Client]] — the other half of the equation
- [[Temporal DV 19 - Testing]] — testing without spinning up real Workers

💡 **Takeaway:** A Worker is a Client-connected process that registers types, polls a Task Queue, and runs work. Keep registration consistent across Workers on the same queue, scale horizontally for throughput, and shut down gracefully.
