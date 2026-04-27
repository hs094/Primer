# RF 01 — SDK Metrics

🔑 **Catalog of metrics emitted by Temporal SDKs (Worker, Workflow, Activity, Client) — what to scrape into Prometheus/StatsD/OTel for [[Temporal DV 18 - Observability]].**

All metrics are prefixed with `temporal_`. Tags shown are the standard labels each metric carries; SDKs may add additional dimensions.

## Activity metrics

| Name | Type | Description |
|---|---|---|
| `temporal_activity_execution_cancelled` | Counter | An Activity Execution was canceled. |
| `temporal_activity_execution_failed` | Counter | An Activity Execution failed (excludes local activities). |
| `temporal_activity_execution_latency` | Histogram | Time from task generation to SDK completion response. |
| `temporal_activity_poll_no_task` | Counter | An Activity Worker poll for an Activity Task timed out. |
| `temporal_activity_schedule_to_start_latency` | Histogram | Time from scheduling to task processing begins. |
| `temporal_activity_succeed_endtoend_latency` | Histogram | Total latency of successfully finished Activity Executions. |
| `temporal_activity_task_error` | Counter | Internal error or panic during Activity Task handling. |

Tags: `activity_type`, `namespace`, `task_queue`, plus `workflow_type` on `temporal_activity_task_error`.

## Local Activity metrics

| Name | Type | Description |
|---|---|---|
| `temporal_local_activity_execution_cancelled` | Counter | A Local Activity was canceled. |
| `temporal_local_activity_execution_failed` | Counter | A Local Activity Execution failed. |
| `temporal_local_activity_execution_latency` | Histogram | Time to complete local activity processing. |
| `temporal_local_activity_succeeded_endtoend_latency` | Histogram | Total latency from scheduling to completion. |
| `temporal_local_activity_total` | Counter | Total number of Local Activity Executions. |

## Workflow metrics

| Name | Type | Description |
|---|---|---|
| `temporal_workflow_cancelled` | Counter | Workflow ended because of a cancellation request. |
| `temporal_workflow_completed` | Counter | A Workflow Execution completed successfully. |
| `temporal_workflow_continue_as_new` | Counter | A Workflow ended with Continue-As-New. |
| `temporal_workflow_endtoend_latency` | Histogram | Total Workflow Execution time from schedule to completion. |
| `temporal_workflow_failed` | Counter | A Workflow Execution failed. |
| `temporal_workflow_active_thread_count` | Gauge | Total amount of Workflow threads in the Worker Process. |

See [[Temporal EN 03 - Workflows]].

## Workflow Task metrics

| Name | Type | Description |
|---|---|---|
| `temporal_workflow_task_execution_failed` | Counter | A Workflow Task failed. Tag `failure_reason`: `NonDeterminismError`, `WorkflowError`. |
| `temporal_workflow_task_execution_latency` | Histogram | Workflow Task Execution time. |
| `temporal_workflow_task_queue_poll_empty` | Counter | A Workflow Worker polled a Task Queue and timed out. |
| `temporal_workflow_task_queue_poll_succeed` | Counter | Workflow Worker polled and successfully picked up a Workflow Task. |
| `temporal_workflow_task_replay_latency` | Histogram | Time to catch up on replaying a Workflow Task. |
| `temporal_workflow_task_schedule_to_start_latency` | Histogram | Schedule-To-Start time of a Workflow Task. |

## Nexus Task metrics

| Name | Type | Description |
|---|---|---|
| `temporal_nexus_poll_no_task` | Counter | A Nexus Worker poll for a Nexus Task timed out. |
| `temporal_nexus_task_schedule_to_start_latency` | Histogram | Time from request arrival to SDK processing begins. |
| `temporal_nexus_task_execution_failed` | Counter | Task handling resulted in error (with `failure_reason`). |
| `temporal_nexus_task_execution_latency` | Histogram | Time from SDK processing start to handler completion. |
| `temporal_nexus_task_endtoend_latency` | Histogram | Total latency of Nexus Tasks from when the request hit the Frontend. |

Extra tags: `nexus_service`, `nexus_operation`.

## Client request metrics

| Name | Type | Description |
|---|---|---|
| `temporal_request` | Counter | Temporal Client made an RPC request. |
| `temporal_request_failure` | Counter | Temporal Client made an RPC request that failed. |
| `temporal_request_latency` | Histogram | Latency of a Temporal Client gRPC request. |
| `temporal_long_request` | Counter | Temporal Client made an RPC long poll request. |
| `temporal_long_request_failure` | Counter | Long poll request failed. |
| `temporal_long_request_latency` | Histogram | Latency of a Temporal Client gRPC long poll request. |

Tags: `namespace`, `operation`.

## Worker metrics

| Name | Type | Description |
|---|---|---|
| `temporal_worker_start` | Counter | A Worker Entity has been registered, created, or started. |
| `temporal_worker_task_slots_available` | Gauge | Total Workflow / Activity / Local Activity / Nexus task execution slots currently available. |
| `temporal_worker_task_slots_used` | Gauge | Total Workflow / Activity / Local Activity / Nexus task execution slots in current use. |
| `temporal_num_pollers` | Gauge | Current number of Worker Entities polling. |
| `temporal_poller_start` | Counter | A Worker Entity poller was started. |

Tags: `namespace`, `task_queue`, `worker_type` (and `poller_type` for `temporal_num_pollers`). See [[Temporal EN 06 - Workers]].

## Sticky cache metrics

| Name | Type | Description |
|---|---|---|
| `temporal_sticky_cache_hit` | Counter | Workflow Task found a cached Workflow Execution. |
| `temporal_sticky_cache_miss` | Counter | Workflow Task did not find a cached Workflow Execution. |
| `temporal_sticky_cache_size` | Gauge | Current cache size, expressed in number of Workflow Executions. |
| `temporal_sticky_cache_total_forced_eviction` | Counter | A Workflow Execution was forced from the cache intentionally. |

## Signals & errors

| Name | Type | Description |
|---|---|---|
| `temporal_corrupted_signals` | Counter | Number of Signals whose payload could not be deserialized. |
| `temporal_unregistered_activity_invocation` | Counter | A request to spawn an Activity that is not registered. |

## Resource metrics

| Name | Type | Description |
|---|---|---|
| `temporal_resource_slots_cpu_usage` | Gauge | CPU usage as a value between 0 and 100. |
| `temporal_resource_slots_mem_usage` | Gauge | Memory usage as a value between 0 and 100. |

⚠️ Gotchas:
- `temporal_activity_execution_failed` excludes **Local Activities** — use `temporal_local_activity_execution_failed` for those.
- `temporal_sticky_cache_miss` rising means workflows are being replayed from scratch — Worker memory likely undersized.
- `temporal_workflow_task_schedule_to_start_latency` is the canonical "Workers can't keep up" signal, not Activity SLT latency.
- `temporal_workflow_task_execution_failed` with `failure_reason=NonDeterminismError` means a deployed code change broke replay — see [[Temporal DV 22 - Debugging]].
- `temporal_long_request_*` are intentionally long-lived (long polls); don't alert on their latency the way you would `temporal_request_*`.

💡 **Takeaway:** Watch `schedule_to_start_latency`, `task_slots_available`, and `sticky_cache_miss` — those three tell you whether your Workers are the bottleneck before users notice.
