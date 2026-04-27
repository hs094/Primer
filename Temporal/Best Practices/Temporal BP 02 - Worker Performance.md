# BP 02 — Worker Performance

🔑 **Actively tune Worker options against your code and runtime — defaults are for development, not production.**

## Quick Checklist

- **Configure Workers appropriately** based on your code, language runtime limits, and system resource constraints.
- **Deploy sufficient Workers** — at least two per Task Queue; monitor and scale to match workload.
- **Separate Task Queues logically** by workload characteristics.
- **Version Workers safely** with Worker Versioning, not code patches.
- **Run benchmarks** under realistic load before trusting any configuration.

## Deployment & Lifecycle

### Packaging
Package Workers as artifacts from CI/CD pipelines. Inject connection parameters via env vars, config files, or CLI args for "more granularity, easier testability, easier upgrades, scalability, and isolation." Multi-stage Docker builds producing minimal production images are the reference pattern.

### Task Queue Separation
Use **separate Task Queues for distinct workloads** — enables independent scaling and prevents resource starvation. Always at least two Workers per queue. Define the Task Queue name as a **constant referenced by both Client and Worker** to avoid mismatches:

```python
ORDER_TQ = "orders-v1"
# client.start_workflow(..., task_queue=ORDER_TQ)
# Worker(client, task_queue=ORDER_TQ, ...)
```

### Worker Versioning
Use Worker Versioning as the default for evolving production Workflow code. It gives "safer rollouts, clearer rollback behavior, and a cleaner operational model" than code patching.

### Event History Hygiene
- Avoid more than several thousand Events per Workflow Execution.
- Use **Continue-As-New** to start a fresh Event History before limits.
- Keep single payloads under 2 MB and total Event History under 50 MB.
- Apply the **Claim Check pattern**: store large data externally (S3, GCS); pass identifiers only.

## Tuning Knobs

| Knob | What it controls |
|---|---|
| Task slots | Concurrent Tasks per Worker — set via Slot Suppliers based on CPU, memory, code demands |
| Sticky cache size | Replay overhead vs. memory consumption |
| Poller counts | Use **Poller Autoscaling** to adjust dynamically |

See [[Temporal CL 11 - worker]] for the CLI surface and [[Temporal DV 15 - Worker Processes]] for the development model.

## Critical Metrics

Track these on every Worker fleet:

- Worker CPU and memory utilization
- `workflow_task_schedule_to_start_latency`
- `activity_task_schedule_to_start_latency`
- `worker_task_slots_available`
- `temporal_long_request_failure`, `temporal_request_failure`
- `temporal_long_request_latency`, `temporal_request_latency`
- `temporal_sticky_cache_size`

Wire these in via [[Temporal RF 01 - SDK Metrics]].

## Interpreting Metrics

| Pattern | Diagnosis | Action |
|---|---|---|
| High Sched-to-Start + High CPU/mem | Workers saturated | Scale up / add Workers |
| High Sched-to-Start + Low CPU/mem | Underutilized | Increase pollers or executor slots |
| Low Sched-to-Start + Low CPU/mem | Over-provisioned | Reduce Workers / slots |

## Cache Optimization

Watch `temporal_sticky_cache_size`. If Workers are memory-bound, **reduce cache size** to allow more concurrent executions. Test sizes in staging before promoting.

## Safe Scale-Down

Before terminating a Worker:
1. Check `worker_task_slots_available` — values near zero mean active Tasks are still running.
2. Use **Graceful Shutdown** so the Worker drains current Tasks before exit.

## ⚠️ Anti-Patterns

⚠️ Running with SDK default Worker options in production.
⚠️ One Worker per Task Queue (no redundancy).
⚠️ Sharing one Task Queue across unrelated workloads — noisy neighbors.
⚠️ Hard-coding Task Queue names separately in Client and Worker (drift waiting to happen).
⚠️ Letting Workflow Event History grow past 50 MB or skipping Continue-As-New.
⚠️ Passing multi-MB payloads through Workflows instead of using the Claim Check pattern.
⚠️ Patching Workflow code in place instead of using Worker Versioning.
⚠️ Killing Workers without graceful shutdown — sticky cache invalidation churn.

## Cross-Refs

- [[Temporal EN 06 - Workers]] — concept
- [[Temporal DV 15 - Worker Processes]] — programming model
- [[Temporal CL 11 - worker]] — CLI
- [[Temporal BP 03 - Pre-Production Testing]] — Worker chaos drills
- [[Temporal BP 06 - Managing APS Limits]] — Worker count vs. APS budget
- [[Temporal SH 05 - Production Checklist]] — pre-launch verification

💡 **Takeaway:** Two Workers per queue, tune slots/pollers/cache to your runtime, watch schedule-to-start latency, and use Versioning + Continue-As-New as defaults.
