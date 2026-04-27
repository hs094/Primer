# SH 04 — Configurable Defaults

🔑 **Temporal ships hard limits on payload size, history size, identifiers, and pending operations. Most are dynamic-config tunables — but raise them carefully, they're sized to protect persistence.**

## Identifiers

Workflow ID, Workflow Type, Task Queue name, Activity ID — all capped at **1000 characters** by default.

```yaml
limit.maxIDLength:
  - value: 1000
    constraints: {}
```

Override via [[Temporal RF 07 - Dynamic Configuration]].

## gRPC Message Size

**4 MB cap per gRPC message received.** Affects single-call payloads (e.g. signal payload, activity input).

## Event History Persistence

Default transaction size for an Event History write: **4 MB**. A single workflow event (input, result, signal payload) cannot exceed this when persisted. See [[Temporal EN 07 - Event History]].

## Payload / Blob Sizes

| Threshold | Default | Behavior |
|---|---|---|
| Warn | **256 KB** | Log: `Blob size exceeds limit` |
| Error | **2 MB** | `ErrBlobSizeExceedsLimit` raised |

Tunable per namespace. Keep payloads small; offload large blobs to external storage with a reference.

## History Size (Per Workflow Execution)

| Threshold | Default | Dynamic key |
|---|---|---|
| Warn | **10 MB** | `HistorySizeLimitWarn` |
| Error | **50 MB** | `HistorySizeLimitError` |

## History Event Count (Per Workflow Execution)

| Threshold | Default |
|---|---|
| Warn | **10,240 events** |
| Error | **51,200 events** |

⚠️ **Both size and event-count caps are hard ceilings.** A long-running workflow that exceeds either will fail. Use Continue-As-New to roll history before you approach the warn threshold.

## Updates Per Execution

- Max in-flight Workflow Updates: **10**
- Max total Updates retained in history: **2000**

## Concurrent Pending Operations

Default cap of **2,000** per workflow execution for each of:

- pending activities
- pending signals
- pending cancellations
- pending child workflows

Tune via:

```yaml
limit.numPendingActivities.error:
  - value: 2000
    constraints: {}
```

Same pattern for `numPendingSignals.error`, `numPendingCancelRequests.error`, `numPendingChildExecutions.error`.

⚠️ **High pending counts amplify history size.** Bumping these without rethinking workflow shape is a path to oversized executions.

## Custom Search Attributes

Per-namespace caps on count and total size. See [[Temporal SH 09 - Self-Hosted Visibility]] and [[Temporal RF 07 - Dynamic Configuration]] for the exact keys.

## Where To Tune

All values above live under [[Temporal RF 07 - Dynamic Configuration]]. Static infra config (shards, persistence) is in [[Temporal RF 06 - Service Configuration]].

⚠️ **Don't raise limits to fix a workflow design problem.** If you're hitting blob/history/event caps, the workflow needs Continue-As-New, smaller payloads, or external blob storage — not a bigger limit.

💡 **Takeaway:** The defaults are guardrails sized to protect persistence — every limit is tunable in dynamic config, but the right move is usually Continue-As-New or smaller payloads, not a higher cap.
