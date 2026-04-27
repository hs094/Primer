# RF 07 — Dynamic Configuration

🔑 **Runtime-tunable knobs the Temporal Service polls without a restart — rate limits, quotas, default policies, and the BLOB / history size guardrails every Workflow inherits.**

Dynamic config lives in a YAML file referenced by `dynamicConfigClient.filepath` (see [[Temporal RF 06 - Service Configuration]]). The Service re-reads it on `pollInterval` (minimum 5s). Defaults below match upstream; production tuning belongs in [[Temporal SH 04 - Configurable Defaults]].

## Frontend Service

| Key | Type | Default | Description |
|---|---|---|---|
| `frontend.rps` | Int | 2400 | Rate limit (requests/second) for requests accepted by each Frontend Service host. |
| `frontend.namespaceRPS` | Int | 2400 | Rate limit (requests/second) per Namespace on the Frontend Service. |
| `frontend.namespaceCount` | Int | 1200 | Concurrent Task Queue polls per Namespace per Frontend host. |
| `frontend.globalNamespaceRPS` | Int | 0 | Rate limit per Namespace, applied across the Cluster. |
| `frontend.persistenceMaxQPS` | Int | 2000 | Max queries/second the Frontend Service can send to the Persistence store. |
| `frontend.persistenceNamespaceMaxQPS` | Int | 0 | Max queries/second per Namespace from Frontend to Persistence. |
| `frontend.visibilityMaxPageSize` | Int | 1000 | Max Workflow Executions returned by `ListWorkflowExecutions` in one page. |
| `frontend.enableServerVersionCheck` | Boolean | true | Enables server to report version info about server and SDK. |

## Internal Frontend Service

| Key | Type | Default | Description |
|---|---|---|---|
| `internal-frontend.globalNamespaceRPS` | Int | 0 | Rate limit per Internal-Frontend host across the Cluster. |

## History Service

| Key | Type | Default | Description |
|---|---|---|---|
| `history.rps` | Int | 3000 | Rate limit (requests/second) per History Service host. |
| `history.persistenceMaxQPS` | Int | 9000 | Max queries/second a History host can send to Persistence. |
| `history.persistenceNamespaceMaxQPS` | Int | 0 | Max queries/second per Namespace from History to Persistence. |
| `history.defaultActivityRetryPolicy` | Map | (defaults) | Server-side default Activity Retry Policy when not explicitly set. |
| `history.defaultWorkflowRetryPolicy` | Map | (defaults) | Default Workflow Retry Policy for unset fields when an explicit policy is provided. |
| `history.maximumSignalsPerExecution` | Int | 10000 | Max Signals a Workflow can receive before raising `Invalid Argument`. |

## Matching Service

| Key | Type | Default | Description |
|---|---|---|---|
| `matching.rps` | Int | 1200 | Rate limit (requests/second) per Matching Service host. |
| `matching.numTaskqueueReadPartitions` | Int | 4 | Number of read partitions for a Task Queue. |
| `matching.numTaskqueueWritePartitions` | Int | 4 | Number of write partitions for a Task Queue. |
| `matching.persistenceMaxQPS` | Int | 9000 | Max queries/second from Matching to Persistence. |
| `matching.persistenceNamespaceMaxQPS` | Int | 0 | Max queries/second per Namespace from Matching to Persistence. |

## Worker Service

| Key | Type | Default | Description |
|---|---|---|---|
| `worker.persistenceMaxQPS` | Int | 100 | Max queries/second a Worker Service host can send to Persistence. |
| `worker.persistenceNamespaceMaxQPS` | Int | 0 | Max queries/second per Namespace from Worker to Persistence. |

(These are server-side internal Workers, not your application [[Temporal EN 06 - Workers]].)

## System / Limits

These define the **hard guardrails** every Workflow Execution inherits.

| Key | Type | Default | Description |
|---|---|---|---|
| `limit.maxIDLength` | Int | 1000 | Length limit for Ids: Namespace, TaskQueue, WorkflowID, ActivityID, TimerID, etc. |
| `limit.blobSize.warn` | Int | 512 KB | BLOB size in an Event when a warning is thrown. |
| `limit.blobSize.error` | Int | 2 MB | BLOB size in an Event when an error occurs. |
| `limit.historySize.warn` | Int | 10 MB | Workflow Event History size that triggers a warning. |
| `limit.historySize.error` | Int | 50 MB | Workflow Event History size that triggers an error. |
| `limit.historyCount.warn` | Int | 10,240 | Workflow Event History event count that triggers a warning. |
| `limit.historyCount.error` | Int | 51,200 | Workflow Event History event count that triggers an error. |
| `limit.numPendingActivities.error` | Int | 2000 | Max pending Activities before `ScheduleActivityTask` fails. |
| `limit.numPendingSignals.error` | Int | 2000 | Max pending Signals a Workflow can hold. |
| `limit.numPendingCancelRequests.error` | Int | 2000 | Max pending requests to cancel other Workflows. |
| `limit.numPendingChildExecutions.error` | Int | 2000 | Max pending Child Workflows. |

These are exactly the limits referenced by the `Pending * Limit Exceeded` rows in [[Temporal RF 04 - Errors]].

## Visibility & Persistence

| Key | Type | Default | Description |
|---|---|---|---|
| `system.visibilityPersistenceMaxReadQPS` | Int | 9000 | Max queries/second the Visibility DB receives for reads. |
| `system.visibilityPersistenceMaxWriteQPS` | Int | 9000 | Max queries/second the Visibility DB receives for writes. |
| `system.enableReadFromSecondaryVisibility` | Boolean | false | Enables reading from the secondary Visibility store. |
| `system.secondaryVisibilityWritingMode` | String | `off` | Enables writing Visibility data to the secondary Visibility store. |

## Nexus

| Key | Type | Default | Description |
|---|---|---|---|
| `system.enableNexus` | Boolean | true | Enables Nexus features. |
| `component.nexusoperations.useSystemCallbackURL` | String | `false` | Whether to use `temporal://system` as the callback URL. |
| `component.nexusoperations.callback.endpoint.template` | String | unset | URL template used to construct Nexus callback URLs. |
| `component.callbacks.allowedAddresses` | Object | unset | Security allow-list of callback URL patterns. |

⚠️ Gotchas:
- **`limit.historySize.error` (50 MB) and `limit.historyCount.error` (51,200) are the canonical reasons to use `Continue-As-New`** — hitting either kills the Workflow.
- The 2,000 pending limits are hard caps per Workflow Execution. They're matched 1:1 with the `Pending * Limit Exceeded` errors.
- `*.persistenceMaxQPS = 0` does **not** mean unlimited — for namespace-scoped variants, 0 disables the per-namespace limit and falls back to the host-level limit.
- `numTaskqueueReadPartitions` / `numTaskqueueWritePartitions` are the lever for very high-throughput Task Queues — bump them carefully and monitor [[Temporal RF 01 - SDK Metrics]] poller counts.
- Defaults assume a small dev cluster — prod clusters routinely raise `frontend.rps`, `history.persistenceMaxQPS`, and `matching.numTaskqueue*Partitions` substantially.
- Changes apply on the next `pollInterval` tick (≥ 5s), not instantly. Don't expect a hot-fix to land in milliseconds.

💡 **Takeaway:** This is the file you edit to raise rate limits, increase Task Queue partitions, and shift Workflow size guardrails — no service restart needed, just a 5-second poll.
