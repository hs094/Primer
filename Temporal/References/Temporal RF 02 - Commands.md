# RF 02 — Commands

🔑 **The fixed vocabulary the SDK uses to tell the Temporal Service what a Workflow wants to do — every Command produces a corresponding Event in the [[Temporal EN 07 - Event History]].**

A Command is the **only** way a Workflow can interact with the world: it cannot perform I/O directly. The SDK collects Commands during a Workflow Task and ships them in the `RespondWorkflowTaskCompleted` RPC. The Service durably appends one Event per accepted Command before the next task runs.

## Lifecycle commands

| Command | Produces | Notes |
|---|---|---|
| `CompleteWorkflowExecution` | `WorkflowExecutionCompleted` | Issued when the Workflow Function returns. One of the few Events that can be the last in a history. |
| `FailWorkflowExecution` | `WorkflowExecutionFailed` | Issued when the Workflow returns an error or raises an unhandled exception. |
| `CancelWorkflowExecution` | `WorkflowExecutionCanceled` | Issued after successful cleanup following a cancellation request. |
| `ContinueAsNewWorkflowExecution` | `WorkflowExecutionContinuedAsNew` | Triggered by Continue-As-New. Closes current run, starts a new run with fresh history. |

See [[Temporal EN 03 - Workflows]] for closure semantics.

## Activity commands

| Command | Produces | Awaitable | Notes |
|---|---|---|---|
| `ScheduleActivityTask` | `ActivityTaskScheduled` | Yes | Schedule an Activity. Limit: **2,000** concurrent pending Activities per Workflow. |
| `RequestCancelActivityTask` | `ActivityTaskCancelRequested` | — | Request cancellation of a previously scheduled Activity. |

See [[Temporal EN 04 - Activities]].

## Timer commands

| Command | Produces | Awaitable | Notes |
|---|---|---|---|
| `StartTimer` | `TimerStarted` | Yes | Initiates a durable timer. |
| `CancelTimer` | `TimerCanceled` | — | Cancels a previously started timer. |

## Child Workflow commands

| Command | Produces | Awaitable | Notes |
|---|---|---|---|
| `StartChildWorkflowExecution` | `ChildWorkflowExecutionStarted` | Yes | Spawns a child workflow. Limit: **2,000** pending children. |

## External Workflow commands

| Command | Produces | Awaitable | Notes |
|---|---|---|---|
| `SignalExternalWorkflowExecution` | `SignalExternalWorkflowExecutionInitiated` | Yes | Send a Signal to another Workflow. Limit: **2,000** pending signals. |
| `RequestCancelExternalWorkflowExecution` | `RequestCancelExternalWorkflowExecutionInitiated` | Yes | Request cancellation of an external Workflow. |

## Nexus commands

| Command | Produces | Awaitable | Notes |
|---|---|---|---|
| `ScheduleNexusOperation` | `NexusOperationScheduled` | Yes | Execute a Nexus Operation. Limit: **30** concurrent operations. |
| `CancelNexusOperation` | `NexusOperationCancelRequested` | — | Cancel a scheduled Nexus operation. |

## SDK / framework commands

| Command | Produces | Notes |
|---|---|---|
| `RecordMarker` | `MarkerRecorded` | Used internally for Local Activities, Side Effects, Versioning markers, etc. |
| `UpsertWorkflowSearchAttributes` | `UpsertWorkflowSearchAttributes` | Update Search Attributes mid-execution (see [[Temporal CL 12 - workflow]]). |
| `ProtocolMessageCommand` | (dynamic) | Helps guarantee ordering constraints for features such as Updates. Event type determined dynamically. |

## Awaitable vs fire-and-forget

"Awaitable" Commands return a future/promise that resolves when the corresponding completion Event arrives:

- `ScheduleActivityTask` → resolves on `ActivityTaskCompleted`/`Failed`/`Canceled`/`TimedOut`.
- `StartTimer` → resolves on `TimerFired`/`TimerCanceled`.
- `StartChildWorkflowExecution` → resolves on `ChildWorkflowExecution*` terminal event.
- `SignalExternalWorkflowExecution` → resolves on `ExternalWorkflowExecutionSignaled` or `Failed`.
- `ScheduleNexusOperation` → resolves on `NexusOperationCompleted`/`Failed`/`Canceled`/`TimedOut`.

Non-awaitable Commands (`CancelTimer`, `RequestCancelActivityTask`, `RecordMarker`, `UpsertWorkflowSearchAttributes`) only emit their initiation Event — there's no acknowledgement to await.

⚠️ Gotchas:
- The 2,000-pending limits are **per Workflow Execution**, enforced server-side. Hitting them produces a `Pending * Limit Exceeded` workflow task failure (see [[Temporal RF 04 - Errors]]).
- Commands are **not** API calls — they're encoded into the WorkflowTaskCompleted response. A Workflow that loops forever issuing Commands without awaiting will fill history and hit `limit.historyCount.error`.
- `ContinueAsNewWorkflowExecution` and `CompleteWorkflowExecution` are mutually exclusive terminal commands — issuing both in one task is a deterministic error.
- `ProtocolMessageCommand` is what makes Updates work; you don't issue it directly from user code.

💡 **Takeaway:** Commands are the contract between SDK and Service — every side effect a Workflow takes must be expressed as one of these ~16 Commands, and each one leaves a permanent fingerprint in the [[Temporal RF 03 - Events]] history.
