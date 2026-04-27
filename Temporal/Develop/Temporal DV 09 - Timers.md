# DV 09 — Timers

🔑 A Workflow can set a **durable Timer** for any fixed time period. In the Python SDK, you just call `asyncio.sleep(seconds)` from inside Workflow code — Temporal converts the call into a Timer event that survives Worker and cluster restarts.

## The API

> "To set a Timer in Python, call the `asyncio.sleep()` function and pass the duration in seconds you want to wait before continuing."

```python
import asyncio

from temporalio import workflow


@workflow.defn
class TimerWorkflow:
    @workflow.run
    async def run(self) -> str:
        # Pause Workflow execution for 10 seconds.
        await asyncio.sleep(10)
        return "done"
```

That's it. There's no special `workflow.sleep` — the standard library coroutine is intercepted by the SDK's deterministic event loop and emitted as a `TimerStarted` history event.

## Why Timers Are Cheap

> "Sleeping is a resource-light operation: it does not tie up the process, and you can run millions of Timers off a single Worker."

A Timer is just a marker in Workflow history plus a server-side wakeup. The Worker process is **not** holding the sleep — it's free to handle other Workflow Tasks while the Timer sits on the cluster.

## Durability

> "Even if your Worker or Temporal Service is down when the time period completes, as soon as your Worker and Temporal Service are back up, the `sleep()` call will resolve and your code will continue executing."

This is the whole point of a durable Timer: it's safe to sleep for **seconds, hours, or months** inside a Workflow. A 90-day waiting period is just `await asyncio.sleep(timedelta(days=90).total_seconds())`.

## Patterns

### Polling with a backoff

```python
@workflow.defn
class PollWorkflow:
    @workflow.run
    async def run(self) -> str:
        for attempt in range(10):
            result = await workflow.execute_activity(
                check_status,
                start_to_close_timeout=timedelta(seconds=5),
            )
            if result == "ready":
                return result
            await asyncio.sleep(2 ** attempt)
        raise ApplicationError("not ready after 10 attempts")
```

### Race a Timer against a Signal

A common pattern: wait for a human Signal, but time out after N hours. Use `asyncio.wait` over the sleep coroutine and a wait-condition on the signal handler — see [[Temporal DV 07 - Message Passing]].

```python
@workflow.defn
class ApprovalWorkflow:
    def __init__(self) -> None:
        self._approved = False

    @workflow.signal
    def approve(self) -> None:
        self._approved = True

    @workflow.run
    async def run(self) -> str:
        try:
            await workflow.wait_condition(
                lambda: self._approved,
                timeout=timedelta(hours=24),
            )
            return "approved"
        except asyncio.TimeoutError:
            return "timed out"
```

⚠️ **Never use `time.sleep()`** inside Workflow code. It blocks the deterministic event loop, doesn't get recorded as a Timer event, and breaks replay. Always use `asyncio.sleep()`.

⚠️ **Never use `datetime.now()` or `time.time()`** for Workflow logic — they're non-deterministic. Use `workflow.now()` / `workflow.time()` if you need the current time as observed by the Workflow.

## See Also

- [[Temporal DV 08 - Schedules]] — recurring executions, not in-Workflow waits.
- [[Temporal DV 02 - Workflow Basics]] — why determinism matters.
- [[Temporal EN 03 - Workflows]] — Timer event semantics.

💡 **Takeaway:** `await asyncio.sleep(seconds)` is the durable Timer. It costs nothing to wait, survives outages, and scales to millions per Worker — but only when you stay within the SDK's deterministic surface and avoid `time.sleep` / wall-clock calls.
