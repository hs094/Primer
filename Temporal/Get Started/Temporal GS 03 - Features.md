# GS 03 — Features

🔑 **Beyond durable execution: timers, retries, signals, queries, schedules, versioning, and observability are first-class.**

## Core primitives
The base layer is [[Temporal EN 03 - Workflows]], [[Temporal EN 04 - Activities]], and [[Temporal EN 06 - Workers]]. Everything else composes on top.

```python
@workflow.defn
class OrderWorkflow:
    @workflow.run
    async def run(self, order_id: str) -> str:
        await workflow.execute_activity(
            charge,
            order_id,
            schedule_to_close_timeout=timedelta(minutes=5),
        )
        return "done"
```

## Failure detection and retries
Every Activity has a configurable Retry Policy: initial interval, backoff coefficient, max attempts, non-retryable error types. Timeouts catch silent hangs:

- `start_to_close_timeout` — single attempt budget
- `schedule_to_close_timeout` — total deadline including retries
- `heartbeat_timeout` — for long-running Activities that report progress

The platform retries failed Activities automatically. Workflow code sees either the success value or — after retries are exhausted — an `ActivityError`.

## Scheduled Workflows
Run a Workflow on a cron string or interval, or at a specific time. The Schedule itself is a durable resource; pause / unpause / backfill are first-class operations.

```bash
temporal schedule create \
  --schedule-id daily-cleanup \
  --workflow-id cleanup \
  --task-queue ops \
  --workflow-type CleanupWorkflow \
  --cron "0 2 * * *"
```

## Cancellation and termination
- **Cancel** — cooperative; Workflow gets a chance to run compensation logic.
- **Terminate** — forced shutdown; no compensation.

Cancel is what you reach for when a long-running process needs to undo prior steps cleanly.

## Message passing: Signals and Queries
**Signals** push data into a running Workflow asynchronously. **Queries** read state synchronously without changing it. **Updates** combine the two: client sends data, Workflow validates and returns a result.

```python
@workflow.defn
class Subscription:
    def __init__(self):
        self.cancelled = False

    @workflow.signal
    async def cancel_sub(self):
        self.cancelled = True

    @workflow.query
    def is_cancelled(self) -> bool:
        return self.cancelled
```

This makes Workflows *reactive* — long-running entities you talk to over their lifetime, not fire-and-forget jobs.

## Versioning
Workflows can run for months. The code that started them may not match what's deployed today. Temporal supports running multiple versions side-by-side via patches / `workflow.patched(...)` so old executions complete on old logic while new ones use new logic.

## Temporal Nexus
Cross-Namespace and cross-service composition primitive. Lets one Workflow call another Workflow as a typed operation across isolation boundaries (different teams, regions, security domains) without leaking implementation details.

## Observability
- **Web UI** — list executions, drill into Event History, inspect pending Activities.
- **Metrics** — SDKs export Prometheus metrics from Workers; Service exports its own.
- **Search Attributes** — index custom fields on Workflow Executions for SQL-like list queries.
- **Dashboards** — wire metrics into Grafana / your stack.

## Testing
Each SDK ships a test framework with a time-skipping in-memory environment. You can write deterministic tests that "wait" for hours of simulated time in milliseconds.

## Data encryption
Plug in a Codec to encrypt payloads end-to-end. The Temporal Service stores opaque ciphertext; only your Workers (and the Web UI configured with the codec) can decrypt.

## Runtime safeguards
The SDK detects common foot-guns at runtime: non-deterministic calls inside a Workflow, oversized payloads, version mismatches. They surface as Workflow Task failures rather than silent corruption.

⚠️ Don't put unbounded loops or unbounded history-growing logic in a single Workflow Execution — Event Histories have size limits. For "forever" processes, use Continue-As-New to start a fresh execution with the carried-over state.

💡 **Takeaway:** Treat the feature list as a menu — pick what your workload needs (retries always; signals for human-in-loop; schedules for cron; versioning the moment you're long-running).
