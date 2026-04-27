# DV 06 — Workflow Timeouts

🔑 **Three workflow timeouts (Execution / Run / Task) and one Retry Policy — all set on `start_workflow` / `execute_workflow`, none enabled by default.**

## The three timeouts

| Timeout | What it bounds |
|---|---|
| **Execution Timeout** (`execution_timeout`) | Total time across the entire workflow chain (including retries and continue-as-new). |
| **Run Timeout** (`run_timeout`) | Time for a single Run (one Run ID). |
| **Task Timeout** (`task_timeout`) | Time a worker has to process a single Workflow Task (default 10s; rarely changed). |

Distinction matters for [[Temporal DV 04 - Continue As New]] — `run_timeout` resets each CAN, `execution_timeout` does not.

## Set timeouts when starting

```python
from datetime import timedelta
from temporalio.client import Client

client = await Client.connect("localhost:7233")

result = await client.execute_workflow(
    YourWorkflow.run,
    "your timeout argument",
    id="your-workflow-id",
    task_queue="your-task-queue",
    # Cap the entire chain
    execution_timeout=timedelta(seconds=2),
    # Cap a single run
    # run_timeout=timedelta(seconds=2),
    # Cap a single Workflow Task
    # task_timeout=timedelta(seconds=2),
)
```

When a timeout fires, the workflow closes with `TIMED_OUT` status; the timeout shows in [[Temporal EN 07 - Event History]] as a `WorkflowExecutionTimedOut` event.

## Retry Policy

> Workflow Executions do not retry by default.

If you want a workflow-level retry (separate from activity retries, which default to "retry forever with backoff"):

```python
from datetime import timedelta
from temporalio.client import Client
from temporalio.common import RetryPolicy

handle = await client.execute_workflow(
    YourWorkflow.run,
    "your retry policy argument",
    id="your-workflow-id",
    task_queue="your-task-queue",
    retry_policy=RetryPolicy(maximum_interval=timedelta(seconds=2)),
)
```

`RetryPolicy` accepts:

- `initial_interval` — first retry backoff (default 1s).
- `backoff_coefficient` — exponential factor (default 2.0).
- `maximum_interval` — cap on backoff.
- `maximum_attempts` — total attempts (0 = unlimited).
- `non_retryable_error_types` — list of exception type names to *not* retry.

Workflow retries fire on uncaught exceptions out of `@workflow.run`. See [[Temporal RF 04 - Errors]] / [[Temporal RF 05 - Failures]] for which errors are retryable.

## Workflow timeouts vs Activity timeouts

Workflow timeouts cap the orchestration; activity timeouts cap each side effect. Activities require *at least one* of `start_to_close_timeout` or `schedule_to_close_timeout` — workflows don't require any. See [[Temporal DV 13 - Activity Timeouts]].

## Choosing values

- **Execution timeout**: use sparingly. A multi-day workflow shouldn't have an execution timeout below several days. Best for end-to-end SLAs.
- **Run timeout**: helpful with continue-as-new — bounds each segment.
- **Task timeout**: leave at default (10s) unless workers are reliably slow processing a single task.

⚠️ **No timeout means no timeout.** A workflow with no execution/run timeout runs forever (or until a worker terminates it). Always pair long-running workflows with [[Temporal DV 04 - Continue As New]] to keep history bounded, even without timeouts.

⚠️ **Workflow retries replay the whole workflow from input.** Unlike activity retries, a workflow retry restarts `@workflow.run` from the top with the original args — internal state is gone. Design idempotency and let activity-level retries handle most failure modes.

💡 **Takeaway:** Set `execution_timeout` for SLA, `run_timeout` for CAN-friendly bounds, leave `task_timeout` alone. `RetryPolicy` is opt-in and restarts `run()` from scratch.
