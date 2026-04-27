# DV 05 — Workflow Cancellation

🔑 **Cancellation is a graceful `SIGTERM` for workflows — surfaced as `asyncio.CancelledError`, giving you a chance to clean up. Termination is `SIGKILL` — no cleanup, no recovery.**

## Cancel vs Terminate

| | Cancel | Terminate |
|---|---|---|
| Recordable | `WorkflowExecutionCancelRequested` event | `WorkflowExecutionTerminated` |
| Cleanup | Yes — workflow gets one more task | No — immediate stop |
| Use for | Normal "please stop" | Misbehaving workflows you want gone |

## Cancel from a client

```python
from temporalio.client import Client

client = await Client.connect("localhost:7233")
await client.get_workflow_handle("your_workflow_id").cancel()
```

To terminate instead:

```python
await client.get_workflow_handle("your_workflow_id").terminate()
```

CLI equivalents in [[Temporal CL 12 - workflow]] (`temporal workflow cancel`, `temporal workflow terminate`).

## Handle cancellation inside a workflow

The cancellation request raises `asyncio.CancelledError` at the next suspension point (`await activity`, `asyncio.sleep`, `wait_condition`, etc.).

```python
from temporalio import workflow
import asyncio

@workflow.defn
class CleanupWorkflow:
    @workflow.run
    async def run(self) -> str:
        try:
            await workflow.execute_activity(
                long_running_step,
                start_to_close_timeout=timedelta(minutes=10),
            )
        except asyncio.CancelledError:
            workflow.logger.info("workflow cancelled, running cleanup")
            await workflow.execute_activity(
                cleanup_step,
                start_to_close_timeout=timedelta(seconds=30),
            )
            raise
        return "ok"
```

The `raise` at the end is required: the workflow only appears as Cancelled in history if `CancelledError` propagates out of `run`. Swallowing it makes the workflow look Completed.

## Cancelling individual activities

`workflow.start_activity(...)` returns a handle whose `.cancel()` requests cancellation of just that activity — the workflow itself keeps running.

```python
activity_handle = workflow.start_activity(
    cancellable_activity, args,
    start_to_close_timeout=timedelta(minutes=5),
    heartbeat_timeout=timedelta(seconds=2),
)
await asyncio.sleep(3)
activity_handle.cancel()
```

## Activity must heartbeat to receive cancellation

```python
from temporalio import activity
import asyncio

@activity.defn
async def cancellable_activity(input: ComposeArgsInput) -> NoReturn:
    try:
        while True:
            print("Heartbeating cancel activity")
            await asyncio.sleep(0.5)
            activity.heartbeat("some details")
    except asyncio.CancelledError:
        print("Activity cancelled")
        raise
```

The activity worker delivers cancellation through `CancelledError` — but only on its next heartbeat. No heartbeat, no cancel. Re-raise to mark the activity as cancelled in history. See [[Temporal DV 11 - Activity Execution]].

## Shielding cleanup work

Cleanup that itself awaits activities is vulnerable: another cancellation can interrupt it. Wrap critical cleanup in `asyncio.shield`:

```python
try:
    await workflow.execute_activity(main_step, ...)
except asyncio.CancelledError:
    await asyncio.shield(
        workflow.execute_activity(cleanup_step, ...)
    )
    raise
```

⚠️ **A swallowed `CancelledError` produces a Completed workflow, not a Cancelled one.** Always re-raise unless you genuinely want to override the cancel.

⚠️ **Activities without heartbeats cannot be cancelled.** They run to their `start_to_close_timeout` no matter how loudly the workflow asks them to stop. Set a `heartbeat_timeout` and call `activity.heartbeat(...)` in any long-running step. See [[Temporal DV 13 - Activity Timeouts]].

⚠️ **Cancellation propagates to children by default.** Child workflows receive cancellation through their own `@workflow.run`. Combine with `parent_close_policy` for finer control — see [[Temporal DV 03 - Child Workflows]].

💡 **Takeaway:** Catch `asyncio.CancelledError`, do your cleanup, re-raise. Activities that need to honour cancellation must heartbeat.
