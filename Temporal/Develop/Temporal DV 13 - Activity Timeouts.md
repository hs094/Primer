# DV 13 — Activity Timeouts

🔑 Every Activity call must specify at least one of three timeouts. They bound different phases of the Execution lifecycle, and `heartbeat_timeout` is what makes long-running Activities crash-detectable. All values are `datetime.timedelta`.

## The Three Timeouts

| Timeout | Bounds |
|---|---|
| **`start_to_close_timeout`** | Max duration of a **single** Activity Task attempt. |
| **`schedule_to_close_timeout`** | Max duration of the **entire** Activity Execution (across all retries). |
| **`schedule_to_start_timeout`** | Max wait from scheduling on the Task Queue until a Worker picks it up. |

⚠️ At least one of `start_to_close_timeout` or `schedule_to_close_timeout` is **required**. The SDK rejects the call otherwise.

## Setting Timeouts

```python
from datetime import timedelta

activity_timeout_result = await workflow.execute_activity(
    your_activity,
    YourParams(greeting, "Activity Timeout option"),
    start_to_close_timeout=timedelta(seconds=10),
    # schedule_to_start_timeout=timedelta(seconds=10),
    # schedule_to_close_timeout=timedelta(seconds=10),
)
```

### Picking the right one

- **Most common**: `start_to_close_timeout` — bounds a single attempt; retries get a fresh budget.
- **Hard deadline**: `schedule_to_close_timeout` — caps total wall-clock including retries.
- **Detect Worker starvation**: `schedule_to_start_timeout` — fail fast if no Worker is polling the Task Queue.

## Retry Policy

Pair timeouts with a `RetryPolicy` to control attempts:

```python
from temporalio.common import RetryPolicy

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

### Override Retry Delay Per-Failure

Raise `ApplicationError` with `next_retry_delay` to dynamically extend the wait:

```python
from datetime import timedelta

from temporalio import activity
from temporalio.exceptions import ApplicationError


@activity.defn
async def my_activity(input: MyActivityInput):
    try:
        # Activity logic
        ...
    except Exception as e:
        attempt = activity.info().attempt
        raise ApplicationError(
            f"Error on attempt {attempt}",
            next_retry_delay=timedelta(seconds=3 * attempt),
        ) from e
```

## Heartbeats

Heartbeats serve two purposes: signal liveness and deliver cancellation. Long-running Activities should heartbeat regularly.

```python
from temporalio import activity


@activity.defn
async def your_activity_definition() -> str:
    activity.heartbeat("heartbeat details!")
    ...
```

The argument is **heartbeat details** — arbitrary serializable state. On retry, the next attempt can read these via `activity.info().heartbeat_details` to resume from the last checkpoint.

## Heartbeat Timeout

Tell the server how often to expect a heartbeat. If silence exceeds this window, the Activity Task is considered failed and gets retried.

```python
workflow.execute_activity(
    activity="your-activity",
    name,
    schedule_to_close_timeout=timedelta(seconds=5),
    heartbeat_timeout=timedelta(seconds=1),
)
```

⚠️ **Without `heartbeat_timeout`, a hung Activity sits unnoticed until `start_to_close_timeout`** — which might be 30 minutes. Set `heartbeat_timeout` short (1–10s) for any Activity that takes longer than a few seconds.

## Choosing Values — Rules of Thumb

- `start_to_close_timeout` should be **~2× your p99 expected duration**. Too short = false retries; too long = slow failure detection.
- `heartbeat_timeout` should be **3–5× your heartbeat cadence**. Heartbeating every 5s? Set timeout to 20s.
- Use `schedule_to_close_timeout` only for hard SLA bounds, otherwise let the `RetryPolicy.maximum_attempts` cap retries.
- `schedule_to_start_timeout` is rarely needed — only when your business logic prefers fast-failing over waiting for a Worker.

## See Also

- [[Temporal DV 11 - Activity Execution]] — call sites.
- [[Temporal DV 14 - Async Activity Completion]] — heartbeating from outside the Worker.
- [[Temporal EN 05 - Detecting Application Failures]] — retry vs. surface.
- [[Temporal RF 04 - Errors]], [[Temporal RF 05 - Failures]] — error type taxonomy.

💡 **Takeaway:** Pick `start_to_close_timeout` per attempt and `heartbeat_timeout` for hung-task detection. Heartbeat with `activity.heartbeat(details)` to checkpoint progress and recover state on retry. Use `RetryPolicy` + `ApplicationError(next_retry_delay=...)` to shape backoff.
