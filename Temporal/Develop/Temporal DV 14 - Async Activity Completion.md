# DV 14 — Async Activity Completion

🔑 Async Activity Completion lets the Activity function **return without finishing** — the result arrives later from an external system via `client.get_async_activity_handle(...).complete(...)`. Use this for human approvals, third-party callbacks, or long-running external jobs where the Worker shouldn't sit idle.

## The Three-Phase Pattern

1. **Activity captures identifying info** — task token, or `(namespace, workflow_id, activity_id)`.
2. **Activity signals async completion** — calls `activity.raise_complete_async()` to tell the Worker "I'm not done; don't auto-complete me."
3. **External system completes** — uses a Client to call `complete()` / `fail()` on the handle.

The Activity Task stays open from Temporal's perspective. Its `start_to_close_timeout` and `heartbeat_timeout` still apply — see [[Temporal DV 13 - Activity Timeouts]].

## Inside the Activity

```python
from temporalio import activity


@activity.defn
async def request_human_approval(payload: ApprovalPayload) -> str:
    captured_token = activity.info().task_token

    # Hand the token to whatever external system handles approvals.
    await send_to_approval_queue(payload, task_token=captured_token)

    # Tell Temporal: don't auto-complete me; someone else will.
    activity.raise_complete_async()
```

`activity.info().task_token` is an opaque `bytes` blob that uniquely identifies this Activity Task attempt server-side.

`activity.raise_complete_async()` raises an internal exception caught by the SDK runtime — it does **not** mark the Activity as failed.

## From the External System

The completer needs a Temporal `Client` and the captured token (or the workflow/activity ID pair):

```python
from temporalio.client import Client


my_client = await Client.connect("localhost:7233")
handle = my_client.get_async_activity_handle(task_token=captured_token)
```

### Complete

```python
await handle.complete("Completion value.")
```

The argument is the Activity's return value, encoded with the configured data converter.

### Fail

```python
await handle.fail(exception)
```

Fails the Activity Task. Temporal applies the configured `RetryPolicy` — so a fail can still be retried unless the exception is non-retryable.

### Heartbeat

```python
await handle.heartbeat(details)
```

Same semantics as `activity.heartbeat()` from inside the Worker — required if you set `heartbeat_timeout`.

### Report Cancellation

```python
await handle.report_cancellation()
```

Acknowledges a cancellation request that came down via heartbeat.

## Alternative: Identify by Workflow + Activity ID

Per the docs, you can identify an Activity using either a "task token" or "a combination of Namespace, Workflow Id, and Activity Id" instead of capturing the token directly:

```python
handle = my_client.get_async_activity_handle(
    workflow_id="approval-workflow-42",
    activity_id="ask-finance-1",
)
```

This avoids passing an opaque blob through your queue/DB — but you must control the Activity ID at call time (set `activity_id=` when starting the Activity).

## Full Pattern: Human Approval

```python
# activities.py
from temporalio import activity


@activity.defn
async def wait_for_approval(req: ApprovalRequest) -> ApprovalDecision:
    token = activity.info().task_token
    await ticket_db.create(
        ticket_id=req.id,
        task_token=token,
        payload=req.dict(),
    )
    activity.raise_complete_async()  # never returns
```

```python
# api.py — FastAPI route a human hits
from fastapi import APIRouter
from temporalio.client import Client

router = APIRouter()


@router.post("/tickets/{ticket_id}/approve")
async def approve(ticket_id: str, decision: ApprovalDecision) -> None:
    ticket = await ticket_db.get(ticket_id)
    client = await Client.connect("localhost:7233")
    handle = client.get_async_activity_handle(task_token=ticket.task_token)
    await handle.complete(decision)
```

## Heartbeating While You Wait

⚠️ Async Completion does **not** suspend the timeouts. If you set `heartbeat_timeout=timedelta(minutes=5)` and the human doesn't respond for an hour, the Activity Task fails. Either:

- Set a long `start_to_close_timeout` (or `schedule_to_close_timeout`) covering the worst-case wait, **and skip `heartbeat_timeout`**.
- Or run a periodic heartbeater that calls `handle.heartbeat(...)` from your external system while waiting.

## See Also

- [[Temporal DV 11 - Activity Execution]] — calling Activities from Workflows.
- [[Temporal DV 12 - Standalone Activities]] — alternative for one-shot jobs.
- [[Temporal DV 13 - Activity Timeouts]] — heartbeat semantics.
- [[Temporal RF 05 - Failures]] — `complete` vs `fail` taxonomy.

💡 **Takeaway:** Capture `activity.info().task_token`, call `activity.raise_complete_async()`, and have the external system finish the job via `client.get_async_activity_handle(task_token=...).complete(...)`. Mind the timeouts — async completion doesn't pause them.
