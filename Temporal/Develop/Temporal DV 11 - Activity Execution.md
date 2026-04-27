# DV 11 — Activity Execution

🔑 Workflows don't run side-effects directly — they spawn **Activity Executions**. The SDK gives you two entry points: `workflow.execute_activity(...)` (await the result) and `workflow.start_activity(...)` (get a handle for manual await/cancel). Every call requires at least one timeout.

## Imports

```python
from datetime import timedelta

from temporalio import workflow
from temporalio.common import RetryPolicy

from your_activities_dacx import your_activity
from your_dataobject_dacx import YourParams
```

## execute_activity()

The shortcut: spawn the Activity, await the result, return.

```python
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

> "`execute_activity()` is a shortcut for `start_activity()` that waits on its result."

## start_activity()

Use when you need the handle (e.g. for cancellation, racing, or fire-and-forget within the Workflow).

```python
handle = workflow.start_activity(
    your_activity,
    YourParams("Hello", name),
    start_to_close_timeout=timedelta(seconds=10),
)
result = await handle
```

> "To get just the handle to wait and cancel separately, use `start_activity()`."

## Required Timeouts

At least one of these **must** be supplied — see [[Temporal DV 13 - Activity Timeouts]] for details:

- `start_to_close_timeout` — max time for one Activity Task attempt.
- `schedule_to_close_timeout` — max time for the entire Activity Execution lifecycle.
- `schedule_to_start_timeout` — max wait between scheduling and a Worker picking it up.

```python
activity_timeout_result = await workflow.execute_activity(
    your_activity,
    YourParams(greeting, "Activity Timeout option"),
    start_to_close_timeout=timedelta(seconds=10),
)
```

⚠️ Forget all three and you'll get a validation error at call time — the SDK refuses to schedule an Activity with no upper bound.

## Retry Policy

Override the default exponential retry with `RetryPolicy`:

```python
activity_result = await workflow.execute_activity(
    your_activity,
    YourParams(greeting, "Retry Policy options"),
    start_to_close_timeout=timedelta(seconds=10),
    retry_policy=RetryPolicy(
        backoff_coefficient=2.0,
        maximum_attempts=5,
        initial_interval=timedelta(seconds=1),
        maximum_interval=timedelta(seconds=2),
    ),
)
```

See [[Temporal EN 05 - Detecting Application Failures]] for which errors are retried vs. surfaced.

## Argument Passing

Activities take **a single positional argument** in the type-safe form. For multi-arg Activities, pass `args=[...]`:

```python
result = await workflow.execute_activity(
    multi_arg_activity,
    args=["greeting", "name"],
    start_to_close_timeout=timedelta(seconds=10),
)
```

> "A single argument to the Activity is positional. Multiple arguments are not supported in the type-safe form of `start_activity()` or `execute_activity()` and must be supplied by the `args` keyword argument."

## Routing to a Different Task Queue

By default Activities run on the Workflow's Task Queue. Override with `task_queue=`:

```python
result = await workflow.execute_activity(
    gpu_intensive_activity,
    payload,
    task_queue="gpu-workers",
    start_to_close_timeout=timedelta(minutes=30),
)
```

This is how you isolate heavy / specialized Workers — see [[Temporal DV 15 - Worker Processes]].

## Idempotency Discipline

Activities **will retry**. Make them idempotent:

- Use the request payload's natural key as the upstream idempotency key.
- For "create resource" calls, check existence first or pass an idempotency-key header.
- Be mindful of payload size — same 2 MB / 4 MB limits as [[Temporal DV 10 - Activity Basics]].

## See Also

- [[Temporal DV 10 - Activity Basics]] — defining `@activity.defn`.
- [[Temporal DV 13 - Activity Timeouts]] — picking the right timeout.
- [[Temporal DV 14 - Async Activity Completion]] — completing from outside the Worker.
- [[Temporal RF 04 - Errors]], [[Temporal RF 05 - Failures]] — error vs. failure semantics.

💡 **Takeaway:** From Workflow code, call `workflow.execute_activity(fn, arg, start_to_close_timeout=...)`. Use `start_activity` when you need a handle, `args=[...]` for multiple arguments, `task_queue=` to route to specialized Workers, and `RetryPolicy` to override the default backoff.
