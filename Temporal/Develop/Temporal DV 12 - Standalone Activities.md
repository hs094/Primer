# DV 12 — Standalone Activities

🔑 Standalone Activities run **directly from a Client**, with no Workflow orchestrating them. Same `@activity.defn` function, same Worker — just enqueued by `client.execute_activity` / `client.start_activity` instead of `workflow.execute_activity`. Useful for one-shot durable jobs that don't need the full Workflow programming model.

> "Standalone Activities are Activities that run independently, without being orchestrated by a Workflow."

⚠️ This API is **pre-release**. Requires Python SDK ≥ 1.23.0 and Temporal CLI v1.6.2-standalone-activity. APIs may change.

## Define the Activity

Identical to any other Activity — see [[Temporal DV 10 - Activity Basics]]:

```python
from dataclasses import dataclass

from temporalio import activity


@dataclass
class ComposeGreetingInput:
    greeting: str
    name: str


@activity.defn
def compose_greeting(input: ComposeGreetingInput) -> str:
    activity.logger.info("Running activity with parameter %s" % input)
    return f"{input.greeting}, {input.name}!"
```

## Worker Setup

Standalone Activities are processed by the same `Worker` you'd use for any other Activity:

```python
from concurrent.futures import ThreadPoolExecutor

from temporalio.client import Client
from temporalio.worker import Worker


client = await Client.connect(target_host="localhost:7233")
worker = Worker(
    client,
    task_queue="my-standalone-activity-task-queue",
    activities=[compose_greeting],
    activity_executor=ThreadPoolExecutor(5),
)
await worker.run()
```

## Execute (Wait for Result)

`client.execute_activity()` durably enqueues the Activity and awaits completion:

```python
activity_result = await client.execute_activity(
    compose_greeting,
    args=[ComposeGreetingInput("Hello", "World")],
    id="my-standalone-activity-id",
    task_queue="my-standalone-activity-task-queue",
    start_to_close_timeout=timedelta(seconds=10),
)
```

Note `id=` is the **Activity ID** here (not a Workflow ID) — this is how you identify the standalone Activity Execution server-side.

## Start (Fire-and-Forget)

`client.start_activity()` enqueues without awaiting:

```python
activity_handle = await client.start_activity(
    compose_greeting,
    args=[ComposeGreetingInput("Hello", "World")],
    id="my-standalone-activity-id",
    task_queue="my-standalone-activity-task-queue",
    start_to_close_timeout=timedelta(seconds=10),
)
```

## Get an Existing Handle

Reattach to a previously started Activity by ID:

```python
activity_handle = client.get_activity_handle(
    activity_id="my-standalone-activity-id",
    run_id="the-run-id",
)
```

## Wait for Result

```python
activity_result = await activity_handle.result()
```

## List and Count

Standalone Activities are visible to the visibility store and queryable with the same List Filter syntax as Workflows:

```python
# List
activities = client.list_activities(
    query="TaskQueue = 'my-standalone-activity-task-queue'"
)
async for info in activities:
    print(f"ActivityID: {info.activity_id}, Status: {info.status}")

# Count
resp = await client.count_activities(
    query="TaskQueue = 'my-standalone-activity-task-queue'"
)
print(f"Total activities: {resp.count}")
```

## When to Use

- Single durable job with retries — no orchestration logic.
- Migrating an existing job-queue worker to Temporal one Activity at a time.
- Background tasks where wrapping in a one-shot Workflow feels heavyweight.

If you need fan-out, child Workflows, Signals, Timers, or branching logic, write a real Workflow instead — see [[Temporal DV 02 - Workflow Basics]] and [[Temporal DV 11 - Activity Execution]].

## See Also

- [[Temporal DV 10 - Activity Basics]] — Activity Definition.
- [[Temporal DV 13 - Activity Timeouts]] — picking timeouts.
- [[Temporal DV 14 - Async Activity Completion]] — completing from outside.
- [[Temporal CL 03 - activity]] — CLI ops on Activities.

💡 **Takeaway:** Standalone Activities give you Temporal's durability and retries with none of the Workflow overhead. Same `@activity.defn`, same Worker, called via `client.execute_activity` / `client.start_activity` with an Activity ID. Pre-release — pin your SDK version.
