# RF 04 — Errors

🔑 **Workflow Task error catalog — the named failure reasons the Service reports when a Workflow Task cannot be processed, plus how to fix each one.**

These appear as `WorkflowTaskFailed` events (see [[Temporal RF 03 - Events]]) and on the Web UI's task error pane. They are distinct from Workflow-level [[Temporal RF 05 - Failures]], which surface to user code.

## Bad Attribute errors

These all mean the SDK sent a [[Temporal RF 02 - Commands]] payload with missing or oversized fields.

| Error | Trigger | Resolution |
|---|---|---|
| `Bad Cancel Timer Attributes` | Missing Timer Id when canceling a Timer. | Add a valid Timer Id and redeploy. |
| `Bad Cancel Workflow Execution Attributes` | Unset CancelWorkflowExecution attributes. | Reset missing attributes and redeploy. |
| `Bad Complete Workflow Execution Attributes` | Unset attributes or payload exceeds size limits. | Reset attributes; reduce payload. |
| `Bad Continue as New Attributes` | Unset/invalid ContinueAsNew attrs; payload or memo too large. | Reset attributes; revalidate Search Attributes. |
| `Bad Fail Workflow Execution Attributes` | Unset FailWorkflowExecution attrs; missing StartToClose/ScheduleToClose timeouts. | Set required timeouts; restart Worker. |
| `Bad Modify Workflow Properties Attributes` | Unset/oversized Memo or payload. | Reset empty attributes; trim Memo/payload. |
| `Bad Record Marker Attributes` | Unset or incorrect Marker name. | Enter valid Marker name and redeploy. |
| `Bad Request Cancel Activity Attributes` | Unset RequestCancelActivity attrs or invalid history builder state. | Update SDK; reset attributes; check for non-deterministic code. |
| `Bad Request Cancel External Workflow Execution Attributes` | Unset/invalid attributes; conflicting Child Workflow Start and RequestCancel. | Reset Workflow Id, Run Id; remove conflicts. |
| `Bad Schedule Activity Attributes` | Unset/invalid ScheduleActivityTask attrs; oversized payload. | Reset attributes; reduce payload. |
| `Bad Schedule Nexus Operation Attributes` | Unset/invalid attrs (e.g., nonexistent Nexus Endpoint). | Inspect error and adjust Endpoint config. |
| `Bad Search Attributes` | Unset/invalid Search Attributes; oversized payload. | Define attributes before retrying; reduce size. |
| `Bad Signal Input Size` | Signal payload exceeds available input size. | Reduce payload size and redeploy. |
| `Bad Signal Workflow Execution Attributes` | Failed validation of SignalExternalWorkflowExecution attrs. | Reset attributes; reduce input size. |
| `Bad Start Child Execution Attributes` | Failed StartChildWorkflowExecution validation; oversized attrs or invalid Search Attributes. | Adjust input; revalidate Search Attributes. |
| `Bad Start Timer Attributes` | Scheduled Event missing Timer Id. | Set valid Timer Id and retry. |

## Cause errors

| Error | Trigger | Resolution |
|---|---|---|
| `Cause Bad Binary` | Worker deployment returned bad binary checksum. | Redeploy with correct binary. |
| `Cause Bad Update` | Workflow attempted completion before receiving Update; missing/invalid Update response. | Verify SDK support; ensure proper Update handling. |
| `Cause Reset Workflow` | Workflow Task failed due to a reset request. | Manually reset Workflow if system hasn't started a new one. |
| `Cause Unhandled Update` | Update received while processing Command that closes the Workflow. | Handle Updates in Workflow code; manage frequency. |
| `Cause Unspecified` | Unknown failure reason. | Examine Workflow Definition for issues. |

## Resource & state errors

| Error | Trigger | Resolution |
|---|---|---|
| `Failover Close Command` | Namespace failover forced Workflow Task to close. | System automatically schedules retry. |
| `Force Close Command` | Workflow Task forced to close. | System schedules retry if recoverable. |
| `gRPC Message Too Large` | Workflow Task response exceeds **4 MB** gRPC limit. | Smaller batches; smaller returns; Continue-As-New; custom Payload Codec compression. |
| `Nondeterminism Error` | Non-deterministic code change in Workflow. | Fix determinism violation. See [[Temporal DV 22 - Debugging]]. |
| `Pending Activities Limit Exceeded` | 2,000 pending Activities reached. | Let current Activities complete. |
| `Pending Child Workflows Limit Exceeded` | 2,000 pending Children reached. | Wait for children to finish. |
| `Pending Nexus Operations Limit Exceeded` | Pending Nexus Operations cap reached. | Complete current operations. |
| `Pending Request Cancel Limit Exceeded` | Pending cancel-request cap reached. | Allow time to drain. |
| `Pending Signals Limit Exceeded` | 2,000 pending Signals reached. | Drain signals before retrying. |
| `Reset Sticky Task Queue` | Sticky Task Queue requires reset. | Reset queue; system retries. |
| `Resource Exhausted – Concurrent Limit` | Concurrent poller count exhausted. | Adjust poller count per Worker. |
| `Resource Exhausted – Persistence Limit` | Persistence rate limit reached. | Reduce persistence load. |
| `Resource Exhausted – RPS Limit` | Workflow exhausted RPS limit. | Reduce request rate. |
| `Resource Exhausted – System Overload` | System overloaded; cannot allocate. | Wait for recovery; reduce load. |
| `Resource Exhausted – Unspecified` | Unknown resource cap. | Investigate capacity. |

## Duplicate & command errors

| Error | Trigger | Resolution |
|---|---|---|
| `Schedule Activity Duplicate Id` | Activity Id already in use. | Use a unique Activity Id and retry. |
| `Start Timer Duplicate Id` | Timer with given Id already started. | Use different Timer Id and retry. |
| `Unhandled Command` | New events available; Workflow attempted closure without handling them; high Signal volume. | Drain Signal channel via `ReceiveAsync`; check logs. |
| `Workflow Worker Unhandled Failure` | Unhandled failure from Workflow Definition. | Debug and fix the Definition. |

⚠️ Gotchas:
- A `WorkflowTaskFailed` does **not** fail the Workflow Execution — the Service retries the task. It will keep retrying until the underlying Worker code is fixed.
- `Nondeterminism Error` is the most common cause of stuck Workflows after a deploy. Versioning APIs or new Task Queues are the cures, not panic-resets.
- "Resource Exhausted" errors are usually a Worker count or [[Temporal RF 07 - Dynamic Configuration]] tuning problem, not user code.
- The 4 MB gRPC limit is **per Workflow Task response**, not per Workflow — many small commands are fine, one giant payload is not.
- These are different from [[Temporal RF 05 - Failures]] which represent business-level Workflow/Activity failures surfaced to user code.

💡 **Takeaway:** If a Workflow appears stuck, read its latest `WorkflowTaskFailed` event — the failure reason maps directly to a row in this table.
