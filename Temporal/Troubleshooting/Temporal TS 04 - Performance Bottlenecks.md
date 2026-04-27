# TS 04 — Performance Bottlenecks

🔑 **Most "Temporal is slow" reports are diagnosable from a handful of SDK metrics — schedule-to-start latency, slot availability, sticky cache, and request latency.**

## Where to start: schedule-to-start latency

The single most useful metric. If it's high, Tasks are sitting in queues longer than they should.

```promql
histogram_quantile(0.95,
  rate(temporal_workflow_task_schedule_to_start_latency_bucket[5m]))

histogram_quantile(0.95,
  rate(temporal_activity_schedule_to_start_latency_bucket[5m]))
```

P95 above ~1s = something is wrong on the Worker side.

Causes:
- Insufficient [[Temporal EN 06 - Workers|Worker]] capacity (CPU / RAM / replicas).
- Misconfigured Worker (too few pollers, too few task slots).
- High Workflow lock latency (excessive Signals on a single Workflow).
- Worker geographically distant from the cluster.
- For Activities specifically: `TaskQueueActivitiesPerSecond` set too low.

Fixes: scale Worker pool, raise pollers/slots ([[Temporal BP 02 - Worker Performance|BP 02]]), co-locate Workers and cluster.

## Workflow end-to-end latency

```promql
histogram_quantile(0.95,
  rate(temporal_workflow_endtoend_latency_bucket[5m]))
```

Total time from start to closure. High values track to:

- Complex Workflows with many steps.
- Frequent retries.
- Worker bottlenecks (see above).
- Slow external dependencies inside Activities.
- Geographic distance.

## Workflow Task execution latency

The time the Worker spends actually running Workflow code per Task. If high:

- CPU-heavy logic in Workflow (move to Activity).
- Slow Local Activities.
- Long replays (huge [[Temporal EN 07 - Event History|Event History]]).
- Resource-starved Worker.
- Blocking calls or accidental infinite loops.
- Heavy Data Converter (e.g. large encrypt/decrypt).

⚠️ Excessive Workflow Task execution latency triggers SDK **deadlock detection** errors. If a Data Converter legitimately needs the time, disable detection only for it:

```go
workflow.DataConverterWithoutDeadlockDetection(myConverter)
```

```java
WorkflowUnsafe.deadlockDetectorOff(() -> { /* ... */ });
```

## Replay latency

```promql
histogram_quantile(0.95,
  rate(workflow_task_replay_latency_bucket[5m]))
```

Healthy is "a few milliseconds." If higher:

- Event History too long → use **Continue-As-New** to reset.
- Heavy Data Converter (encryption-on-replay overhead).
- Big payloads.
- Sticky cache evicting too much.

## Sticky cache health

Three metrics, one story:

```promql
temporal_sticky_cache_size                       # current cached executions
rate(temporal_sticky_cache_hit_total[5m])
rate(temporal_sticky_cache_miss_total[5m])
rate(temporal_sticky_cache_total_forced_eviction_total[5m])
```

- High **hit rate** + low **miss rate** = sticky scheduling working; only new events transmitted on each Workflow Task.
- High **forced eviction** = cache too small for working set. Bump `WorkflowCacheSize` (default Go SDK = 10,000) if Worker memory has headroom.

⚠️ Bigger cache → more memory. Don't blindly 10x it on a 1 GB pod.

## Task slot saturation

```promql
temporal_worker_task_slots_available{worker_type="WorkflowWorker"}
temporal_worker_task_slots_available{worker_type="ActivityWorker"}
```

When this approaches 0, the Worker can't accept new Tasks.

**Workflow Worker depletion:**
- Volume past current `MaxConcurrentWorkflowTaskExecutionSize`.
- Long-running Workflow Tasks (CPU-heavy code).
- Fix: scale horizontally first; raise concurrency only if Worker is not CPU/MEM bound.

**Activity Worker depletion:**
- Blocked / zombie Activities (timeouts firing without Heartbeats killing them).
- Activity code blocking on slow downstream services with no client-side timeout.
- Fix: client-side HTTP/DB timeouts inside Activities, raise concurrency, scale Workers.

## Activity execution latency

```promql
histogram_quantile(0.95,
  rate(temporal_activity_execution_latency_bucket{activity_type="charge_card"}[5m]))
```

Almost always reflects the Activity body, not Temporal:

- Slow third-party API.
- Worker CPU saturated (under-provisioned).
- Network latency to an external dependency.

Tune `(Max)ConcurrentActivityExecutionSize` and `(Max)WorkerActivitiesPerSecond` if many fast Activities are queueing behind a few slow ones.

## Network / RPC failures

```promql
rate(temporal_long_request_failure_total[5m])         # poll calls
rate(temporal_request_failure_total[5m])              # all RPCs
histogram_quantile(0.95, rate(temporal_request_latency_bucket[5m]))
```

`temporal_long_request_failure_total` covers `PollWorkflowTaskQueue`, `PollActivityTaskQueue`, `GetWorkflowExecutionHistory`. High values mean network issues, rate limiting (`ResourceExhausted`), or server problems — pair with [[Temporal TS 02 - Deadline Exceeded Error|TS 02]].

`temporal_request_failure_total` includes blob-size violations ([[Temporal TS 01 - Blob Size Limit Error|TS 01]]) and signaling closed Workflows.

## Polling rate

```promql
rate(temporal_long_request_total{operation="PollActivityTaskQueue"}[1m])
rate(temporal_long_request_total{operation="PollWorkflowTaskQueue"}[1m])
```

Per-second polling rate. Useful as a **load gauge** and to confirm Workers are actually polling (not silently stuck).

## Tuning order (do these in order, not all at once)

1. **Scale Worker replicas** — cheapest, safest, fixes most schedule-to-start spikes.
2. **Raise pollers** — pollers cap how many Tasks can be in-flight per Worker.
3. **Raise task slots** — only if Worker has CPU/MEM to spare.
4. **Cache size** — raise if forced eviction is high *and* memory is available.
5. **Activity concurrency / per-second caps** — last, after upstream dependencies are sized correctly.

⚠️ Cranking everything at once hides which knob actually mattered and risks OOMing the Worker.

## Footguns

⚠️ Looking at Worker CPU only. A Worker can be 30% CPU and still saturated on slots.
⚠️ Treating high Activity latency as a Temporal problem when it's a downstream API problem.
⚠️ Letting Event History grow unbounded. Use **Continue-As-New** before replay latency dominates.
⚠️ Disabling deadlock detection broadly to "fix" slow Workflow Tasks — fix the code instead.
⚠️ Operating without dashboards. SDK + Service metrics together are non-negotiable in prod ([[Temporal SH 08 - Monitoring|SH 08]]).

Related: [[Temporal BP 02 - Worker Performance|BP 02]] (worker tuning), [[Temporal DV 15 - Worker Processes|DV 15]] (worker process model), [[Temporal TS 02 - Deadline Exceeded Error|TS 02]] (overload), [[Temporal TS 01 - Blob Size Limit Error|TS 01]] (blob limits), [[Temporal RF 05 - Failures|RF 05]] (failure model).

💡 **Takeaway:** Diagnose with schedule-to-start latency, slot availability, and cache eviction first; scale Workers before tuning concurrency knobs; payload size and external services usually explain the rest.
