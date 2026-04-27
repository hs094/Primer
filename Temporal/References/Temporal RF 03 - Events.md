# RF 03 — Events

🔑 **The complete vocabulary of Events the Temporal Service writes into a Workflow's [[Temporal EN 07 - Event History]] — replaying them deterministically reconstructs Workflow state.**

Each Event has a unique `event_id`, `event_time`, `event_type`, and a typed attributes payload. Events are appended in monotonically increasing order by the History Service. Most are caused by [[Temporal RF 02 - Commands]] from a Worker; some by external requests (signals, cancels, terminations); some by the Service itself (timer fires, task scheduled, timeouts).

## Workflow Execution events

| Event | Description | Key fields |
|---|---|---|
| `WorkflowExecutionStarted` | First Event in any history. | `workflow_type`, `task_queue`, `input`, `workflow_execution_timeout`, `retry_policy` |
| `WorkflowExecutionCompleted` | Workflow finished successfully. | `result`, `workflow_task_completed_event_id`, `new_execution_run_id` |
| `WorkflowExecutionFailed` | Workflow finished with error. | `failure`, `retry_state`, `workflow_task_completed_event_id` |
| `WorkflowExecutionTimedOut` | Workflow exceeded its timeout. | `retry_state`, `new_execution_run_id` |
| `WorkflowExecutionCancelRequested` | Cancellation requested by external party. | `cause`, `external_workflow_execution`, `identity` |
| `WorkflowExecutionCanceled` | Workflow successfully canceled. | `workflow_task_completed_event_id`, `details` |
| `WorkflowExecutionSignaled` | Workflow received a Signal. | `signal_name`, `input`, `identity`, `header` |
| `WorkflowExecutionTerminated` | Workflow forcefully stopped. | `reason`, `details`, `identity` |
| `WorkflowExecutionContinuedAsNew` | New run started in the same logical chain. | `new_execution_run_id`, `workflow_type`, `input`, `backoff_start_interval` |
| `WorkflowExecutionOptionsUpdated` | Configuration parameters changed mid-flight. | `versioning_override`, `attached_completion_callbacks` |

## Workflow Task events

| Event | Description | Key fields |
|---|---|---|
| `WorkflowTaskScheduled` | Task placed on the Task Queue. | `task_queue`, `start_to_close_timeout`, `attempt` |
| `WorkflowTaskStarted` | A Worker picked up the task. | `scheduled_event_id`, `identity`, `request_id` |
| `WorkflowTaskCompleted` | Worker returned commands successfully. | `scheduled_event_id`, `started_event_id`, `identity`, `binary_checksum` |
| `WorkflowTaskTimedOut` | Task processing exceeded its timeout. | `scheduled_event_id`, `started_event_id`, `timeout_type` |
| `WorkflowTaskFailed` | Task encountered an error (often non-determinism or reset). | `failure`, `identity`, `base_run_id`, `new_run_id`, `binary_checksum` |

See [[Temporal EN 06 - Workers]] for how tasks are dispatched.

## Activity Task events

| Event | Description | Key fields |
|---|---|---|
| `ActivityTaskScheduled` | Activity queued. | `activity_id`, `activity_type`, `task_queue`, `input`, `schedule_to_close_timeout` |
| `ActivityTaskStarted` | Worker began executing. | `scheduled_event_id`, `identity`, `request_id`, `attempt`, `last_failure` |
| `ActivityTaskCompleted` | Activity returned successfully. | `result`, `scheduled_event_id`, `started_event_id`, `identity` |
| `ActivityTaskFailed` | Activity raised an exception. | `failure`, `scheduled_event_id`, `started_event_id`, `retry_state` |
| `ActivityTaskTimedOut` | Exceeded one of the timeouts. | `failure`, `scheduled_event_id`, `started_event_id`, `timeout_type` |
| `ActivityTaskCancelRequested` | Workflow asked to cancel. | `scheduled_event_id`, `workflow_task_completed_event_id` |
| `ActivityTaskCanceled` | Activity confirmed cancellation. | `details`, `scheduled_event_id`, `started_event_id`, `identity` |

See [[Temporal EN 04 - Activities]].

## Timer events

| Event | Description | Key fields |
|---|---|---|
| `TimerStarted` | Timer created. | `timer_id`, `start_to_fire_timeout`, `workflow_task_completed_event_id` |
| `TimerFired` | Timer expired. | `timer_id`, `started_event_id` |
| `TimerCanceled` | Timer canceled before firing. | `timer_id`, `started_event_id`, `workflow_task_completed_event_id` |

## External Workflow events

| Event | Description |
|---|---|
| `RequestCancelExternalWorkflowExecutionInitiated` | Cancellation request sent to external Workflow. |
| `RequestCancelExternalWorkflowExecutionFailed` | Server could not deliver cancel. |
| `ExternalWorkflowExecutionCancelRequested` | Server successfully relayed cancellation. |
| `SignalExternalWorkflowExecutionInitiated` | Signal request sent. |
| `SignalExternalWorkflowExecutionFailed` | Server could not deliver signal. |
| `ExternalWorkflowExecutionSignaled` | Signal successfully delivered. |

## Child Workflow events

| Event | Description |
|---|---|
| `StartChildWorkflowExecutionInitiated` | Parent requested a child. |
| `StartChildWorkflowExecutionFailed` | Child could not be started. |
| `ChildWorkflowExecutionStarted` | Child is running. |
| `ChildWorkflowExecutionCompleted` | Child finished successfully. |
| `ChildWorkflowExecutionFailed` | Child finished with error. |
| `ChildWorkflowExecutionCanceled` | Child confirmed cancellation. |
| `ChildWorkflowExecutionTimedOut` | Child exceeded its timeout. |
| `ChildWorkflowExecutionTerminated` | Child was terminated. |

Common fields: `namespace`, `workflow_execution`, `workflow_type`, `initiated_event_id`, `started_event_id`, `result`/`failure`/`details`, `retry_state`.

## Marker & Search Attribute events

| Event | Description | Key fields |
|---|---|---|
| `MarkerRecorded` | Custom marker (Local Activities, Side Effects, Versioning). | `marker_name`, `details`, `workflow_task_completed_event_id`, `failure` |
| `UpsertWorkflowSearchAttributes` | Search Attributes synchronized. | `workflow_task_completed_event_id`, `search_attributes` |

## Workflow Update events

| Event | Description | Key fields |
|---|---|---|
| `WorkflowExecutionUpdateAcceptedEvent` | Workflow accepted an Update request. | `protocol_instance_id`, `accepted_request_message_id`, `accepted_request` |
| `WorkflowExecutionUpdateCompletedEvent` | Update finished executing. | `meta`, `accepted_event_id`, `outcome` |

## Nexus Operation events

| Event | Description | Key fields |
|---|---|---|
| `NexusOperationScheduled` | Operation initiated. | `endpoint`, `service`, `operation`, `input`, `schedule_to_close_timeout`, `request_id` |
| `NexusOperationStarted` | Asynchronous handler responded. | `scheduled_event_id`, `operation_token`, `request_id` |
| `NexusOperationCompleted` | Operation finished successfully. | `scheduled_event_id`, `result`, `request_id` |
| `NexusOperationFailed` | Handler returned error. | `scheduled_event_id`, `failure`, `request_id` |
| `NexusOperationTimedOut` | Operation exceeded schedule-to-close. | `scheduled_event_id`, `failure`, `request_id` |
| `NexusOperationCancelRequested` | Workflow requested cancellation. | `scheduled_event_id`, `workflow_task_completed_event_id` |
| `NexusOperationCanceled` | Operation confirmed cancellation. | `scheduled_event_id`, `failure`, `request_id` |

⚠️ Gotchas:
- Most events reference `workflow_task_completed_event_id` — this is what links them back to the Workflow Task that produced them. If you see no such id, the event was service-originated (signals, cancel requests, timer fires).
- `WorkflowTaskTimedOut` is often a Worker problem (no Worker available, or stuck in user code) — see [[Temporal TS 02 - Deadline Exceeded Error]].
- `WorkflowExecutionContinuedAsNew` is **not** a failure — it's the normal way long-running Workflows reset history.
- The terminal events (`Completed`, `Failed`, `Canceled`, `TimedOut`, `Terminated`, `ContinuedAsNew`) are mutually exclusive — only one can be the last in a history.
- `MarkerRecorded` is how Local Activities and Side Effects appear in history — don't confuse a Local Activity result with `ActivityTaskCompleted`.
- `WorkflowTaskFailed` does **not** end the Workflow — the Service schedules another Workflow Task and retries.

💡 **Takeaway:** Every observable thing a Workflow does ends up here. When debugging, read the history top-to-bottom — the next event after a Command's initiation event tells you the outcome.
