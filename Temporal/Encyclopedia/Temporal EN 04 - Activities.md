# EN 04 — Activities

🔑 **An Activity is a single, well-defined unit of work — the only place I/O lives — automatically retried on failure.**

## Definition

An Activity is *a normal function or method that executes a single, well-defined action (either short or long running), such as calling another service, transcoding a media file, or sending an email message*.

Unlike Workflow code, Activity code:
- **Can be non-deterministic.**
- **Should be idempotent** (retries will re-invoke it).
- Performs all I/O, randomness, and wall-clock work in a Temporal app.

## Common Use Cases

- Single write operations (update user, charge a card).
- Batches of similar writes (create N orders, send N notifications).
- Read-then-write sequences (check status before update).
- Memoized reads (LLM calls, large downloads, slow polls).

## Core Concepts

| Term | Meaning |
|---|---|
| **Activity Definition** | The function/method registered with the Worker. |
| **Activity Type** | Name that maps to a Definition. |
| **Activity Execution** | A logical run of an Activity from schedule through final close (across retries). |
| **Activity Task** | A single attempt at running the Activity, dispatched on a Task Queue. |
| **Activity ID** | Per-execution identifier within a Workflow. |

## Execution Model

```
Workflow code:                  Worker:                   Service:
  ExecuteActivity ──Command──▶  picks up Task ──result──▶ writes
                                                          ActivityTaskCompleted
                                                          to Event History
```

Workflows don't run Activities — they *request* them. The request becomes a Command, the Service schedules an Activity Task on a Task Queue, a Worker picks it up, runs it, and posts back the result. Failure → retry per policy.

## Timeouts (set at scheduling time)

Configure on the Workflow side when calling the Activity:

- **Schedule-To-Start** — task in queue → worker picks it up. Default infinite. Detects worker fleet starvation. *Does not* trigger retries.
- **Start-To-Close** — single attempt's max runtime. **Always set this.** Required for the Service to force retries when a worker silently dies.
- **Schedule-To-Close** — outer bound across all retries (schedule → final close). Default infinite. Caps total wall-time independently of `MaximumAttempts`.
- **Heartbeat** — max gap between heartbeats. For long-running activities only. If exceeded, attempt fails and retries.

See [[Temporal EN 05 - Detecting Application Failures]].

## Heartbeats

Long-running Activities call `heartbeat(details)` periodically to:
1. Prove liveness (so Heartbeat Timeout doesn't fire).
2. Checkpoint progress — `details` survive into the next retry attempt for resumption.

Heartbeats are throttled by the SDK; don't worry about call frequency.

## Retries

Activities have a **default Retry Policy** and auto-retry on failure. Workflows don't (workflows must be deterministic). See [[Temporal RF 05 - Failures]].

Default policy:

| Field | Default |
|---|---|
| Initial Interval | 1s |
| Backoff Coefficient | 2.0 |
| Maximum Interval | 100× Initial (100s) |
| Maximum Attempts | unlimited |
| Non-Retryable Errors | none |

Mark errors non-retryable (e.g. `InvalidArgument`, `BusinessRuleViolated`) to stop retry loops on permanent failures.

## Cancellation & Async Completion

- **Cancellation** — workflow can request cancel; activity sees it via heartbeat response and should clean up + raise `CancelledError`.
- **Async Completion** — activity returns control immediately, work happens elsewhere (queue, webhook). Complete via `client.complete_activity` from outside the worker. Required for human-in-the-loop or long external waits.

## Local vs Standalone

- **Activity** (regular) — scheduled on a Task Queue; dispatched by the Service; persisted in Event History per attempt. Survives worker death.
- **Local Activity** — runs in-process on the same Worker as the Workflow Task; lower latency, no Task Queue dispatch; persisted only on completion. Use for short, low-stakes work.
- **Standalone Activity** — invoked outside a Workflow context.

## Cross-Refs

- [[Temporal DV 10 - Activity Basics]] — developer how-to
- [[Temporal EN 03 - Workflows]] — caller side
- [[Temporal EN 06 - Workers]] — what executes Activity Tasks
- [[Temporal EN 07 - Event History]] — `ActivityTaskScheduled/Started/Completed/Failed/TimedOut/Cancelled`

⚠️ **Always set `Start-To-Close`.** Without it, a wedged worker holds an Activity forever — the Service has no way to declare it dead and reassign it.

⚠️ Idempotency is your problem. A retry after partial success will run your code again; design for it (idempotency keys, upsert semantics).

💡 **Takeaway:** Activities are where reality happens. Set Start-To-Close, make them idempotent, heartbeat the long ones, mark permanent errors non-retryable.
