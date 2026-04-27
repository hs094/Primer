# RF 05 — Failures

🔑 **The typed failure hierarchy that propagates through Workflows, Activities, Child Workflows, and Nexus operations — what user code actually catches.**

Failures are the user-facing side of error handling: each maps to a typed proto and is delivered to the calling Workflow as the `cause` of an [[Temporal RF 03 - Events]] like `ActivityTaskFailed`. They differ from [[Temporal RF 04 - Errors]] which are internal Workflow Task processing problems.

## Type hierarchy

| Failure | Role |
|---|---|
| `TemporalFailure` | Base class. Common fields on every subtype. |
| `ApplicationFailure` | Only failure type **created and thrown by user code**. |
| `CancelledFailure` | Successful cancellation of Workflow / Activity / Nexus operation. |
| `ActivityFailure` | Wraps the cause when an Activity fails — delivered to the parent Workflow. |
| `ChildWorkflowFailure` | Wraps the cause when a Child Workflow fails. |
| `NexusOperationFailure` | Wraps the cause when a Nexus operation fails. |
| `TimeoutFailure` | Activity or Workflow exceeded a configured timeout. |
| `TerminatedFailure` | Used as `cause` when a Workflow is terminated. |
| `ServerFailure` | Errors that originate in the Temporal Service (infrastructure). |

## Common fields (`TemporalFailure`)

| Field | Description |
|---|---|
| `message` | Human-readable failure message. |
| `stack_trace` | Captured stack trace from the throwing process. |
| `source` | SDK origin (e.g., `GoSDK`, `JavaSDK`, `PythonSDK`). |
| `cause` | Underlying failure that caused this one (chained). |
| `encoded_attributes` | Optional encrypted/encoded extra attributes. |

## ApplicationFailure

The only failure user code is allowed to construct. Use it to communicate domain-specific issues from Workflows, Activities, and Nexus operations.

| Field | Description |
|---|---|
| `type` | Free-form string for categorization (e.g., `"InsufficientFunds"`). |
| `message` | Human description. |
| `non_retryable` | If `true`, the Activity/Workflow Retry Policy will **not** retry. Defaults to `false`. |
| `details` | Arbitrary structured payload(s). |

### Sub-flavors

- **Workflow Task Failure** — non-Temporal exceptions raised during Workflow processing. Causes Workflow Task **retries** until execution timeout (it does NOT fail the Workflow Execution).
- **Workflow Execution Failure** — an `ApplicationError` that **terminates** the Workflow and prevents further attempts.
- **Activity Failure** — thrown from an Activity. Converted error sets `type`, `message`, `non_retryable` (false by default), `cause`.
- **Nexus Operation Task Failure** — unexpected error processing a Nexus task; auto-retries.
- **Nexus Operation Execution Failure** — non-retryable application failure that terminates the operation.

## CancelledFailure

Represents a **successful** cancellation. Appears as the `cause` of:

- `ActivityFailure` when an Activity confirms cancellation.
- `ChildWorkflowFailure` when a Child Workflow is canceled.
- `NexusOperationFailure` when a Nexus operation is canceled.

Catching `CancelledFailure` is how Workflow code distinguishes cancellation from a real error.

## ActivityFailure

Delivered to the Workflow Execution when an [[Temporal EN 04 - Activities|Activity]] fails. It's a **wrapper**:

| Field | Description |
|---|---|
| `scheduled_event_id` | Event id of the `ActivityTaskScheduled`. |
| `started_event_id` | Event id of the `ActivityTaskStarted`. |
| `identity` | Worker identity that ran the activity. |
| `activity_type` | Activity name. |
| `activity_id` | Activity id. |
| `retry_state` | Why retries stopped (e.g., `MaximumAttemptsReached`, `NonRetryableFailure`). |
| `cause` | The actual underlying `ApplicationFailure` / `TimeoutFailure` / `CancelledFailure`. |

Always inspect `.cause` — `ActivityFailure` itself is just metadata.

## ChildWorkflowFailure

Wraps the failure of a Child Workflow (see [[Temporal EN 03 - Workflows]] parent/child). Fields mirror `ActivityFailure` but for Workflow metadata.

## NexusOperationFailure

Indicates a failed Nexus operation.

| Field | Description |
|---|---|
| `endpoint` | Nexus endpoint name. |
| `service` | Nexus service name. |
| `operation` | Operation name. |
| `operation_token` | Token for async operations. |
| `nexus_error_code` | Nexus protocol error code. |
| `cause` | Underlying failure. |

## TimeoutFailure

Activity or Workflow exceeded a timeout.

| Field | Description |
|---|---|
| `timeout_type` | `START_TO_CLOSE`, `SCHEDULE_TO_START`, `SCHEDULE_TO_CLOSE`, or `HEARTBEAT`. |
| `last_heartbeat_details` | Details from the most recent heartbeat (Activities only). |
| `cause` | Optional cause if available. |

See [[Temporal TS 02 - Deadline Exceeded Error]].

## TerminatedFailure

Used as `cause` when a Workflow is terminated — visible to a parent Workflow handling a Child's result, or to a client retrieving a terminated Workflow's result. Carries `reason` and `details`.

## ServerFailure

Originates in the Temporal Service itself — infrastructure-level issues. Generally retried by the Service; user code rarely needs special handling beyond noting `non_retryable` semantics.

⚠️ Gotchas:
- **`ApplicationFailure` is the only one user code creates.** Constructing `ActivityFailure` directly is wrong — let the SDK wrap your error.
- A **Workflow Task Failure** (a Python/Go/Java exception raised in Workflow code) is **not** the same as a **Workflow Execution Failure** — the former retries indefinitely; the latter ends the Workflow. Throw `ApplicationFailure` to fail the Workflow execution.
- `non_retryable=True` on an `ApplicationFailure` skips the Activity/Workflow Retry Policy — use it for true business errors (e.g., validation failure), never for transient I/O.
- When catching, **always inspect `.cause`** — top-level types like `ActivityFailure` and `ChildWorkflowFailure` are wrappers.
- `CancelledFailure` is a *success path* — distinguish it from `ApplicationFailure` in your catch blocks or you'll log spurious errors.

💡 **Takeaway:** Throw `ApplicationFailure` from your code, expect `ActivityFailure` / `ChildWorkflowFailure` wrappers in the caller, and always unwrap `.cause` to find the real reason.
