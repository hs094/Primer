# EN 07 — Event History

🔑 **The Event History is the per-Workflow append-only log that makes durable execution work — every Command becomes an Event, every replay reads the log.**

## Definition

The Event History is *a complete and durable log of everything that has happened in the lifecycle of a Workflow Execution*. It is the source of truth for replay, the basis for visibility, and the substrate that lets workflows survive crashes and span years.

## Command/Event Handshake

Workflow code does not perform actions directly. Instead:

```
Workflow code ── ExecuteActivity ──▶ Command ──▶ Service ──▶ Event ──▶ History
                                                    │
                                              schedule task
                                                    ▼
                                                  Worker
                                                    │
                                                 result
                                                    ▼
                                          Event appended to History
```

- **Command** — *a requested action issued by a Worker to the Temporal Service after a Workflow Task Execution completes.*
- **Event** — durable, persisted record. The Service converts Commands → Events and appends them.

When a Worker fails or rebalances, the next Worker re-runs the Workflow Definition against the History. Each replayed Command must match the corresponding Event; mismatch → non-determinism error.

## Common Event Types

Lifecycle:
- `WorkflowExecutionStarted`
- `WorkflowExecutionCompleted`
- `WorkflowExecutionFailed`
- `WorkflowExecutionTimedOut`
- `WorkflowExecutionCanceled`
- `WorkflowExecutionTerminated`
- `WorkflowExecutionContinuedAsNew`
- `WorkflowExecutionSignaled`

Workflow Tasks:
- `WorkflowTaskScheduled`
- `WorkflowTaskStarted`
- `WorkflowTaskCompleted`
- `WorkflowTaskFailed`
- `WorkflowTaskTimedOut`

Activity Tasks:
- `ActivityTaskScheduled`
- `ActivityTaskStarted`
- `ActivityTaskCompleted`
- `ActivityTaskFailed`
- `ActivityTaskTimedOut`
- `ActivityTaskCancelRequested`
- `ActivityTaskCanceled`

Timers:
- `TimerStarted`
- `TimerFired`
- `TimerCanceled`

Child Workflows:
- `StartChildWorkflowExecutionInitiated`
- `ChildWorkflowExecutionStarted`
- `ChildWorkflowExecutionCompleted`
- `ChildWorkflowExecutionFailed`

Misc:
- `MarkerRecorded` (Side Effects, Versioning, Local Activities)
- `UpsertWorkflowSearchAttributes`

See [[Temporal RF 03 - Events]] for the full enumeration.

## Why Replay Works

```
Initial state                History           In-memory state after replay
─────────────             ─────────────         ──────────────────────────
   ∅           +     [evt1, evt2, ...]   =     same state every time
                                                (because workflow code is deterministic)
```

Two invariants make this safe:
1. **Deterministic workflow code** — same input + same history → same Commands.
2. **Append-only History** — events never mutate, never reorder.

Together: any Worker can pick up where any other left off.

## History Limits

The History has hard ceilings (per the Service):

- **51,200 events** soft warning, hard cap.
- **50 MB** total history size.

Workflow types that loop forever or accumulate huge state will trip these. The escape hatch is **Continue-As-New**.

## Continue-As-New

Closes the current Workflow Run with state `ContinuedAsNew` and starts a new Run with a fresh, empty History — same Workflow ID, new Run ID. Carry-over state passes as input to the new run.

```
Run 1 history = 50,000 events
       │
       ├─ ContinueAsNew(carry_state) ──▶  Run 2 history = 1 event
       │
                                          (chain visible via Workflow ID)
```

Use when:
- Long-running entity workflows (per-user, per-account state machines).
- Cron-style loops that accumulate iterations.
- Approaching the event/size cap.

## Replay & Stickiness

After a successful Workflow Task, the Service prefers the same Worker for the next task (Sticky Execution) — that Worker's in-memory state is already hot, so it skips a full replay. Cold start (new Worker, restart, rebalance) → full replay from event 1. See [[Temporal EN 06 - Workers]].

## Tooling

- `temporal workflow show --workflow-id <id>` — print the History.
- Web UI history view — chronological event stream with payloads.
- Programmatic: `client.get_workflow_handle(...).fetch_history()`.

## Cross-Refs

- [[Temporal EN 03 - Workflows]] — caller of every Command
- [[Temporal EN 06 - Workers]] — replays the History
- [[Temporal EN 04 - Activities]] — generates `ActivityTask*` events
- [[Temporal RF 03 - Events]] — full event reference
- [[Temporal CL 12 - workflow]] — `workflow show`

⚠️ Adding/removing/reordering Command-producing API calls in deployed workflow code breaks replay for in-flight executions. Use Patching / Worker Versioning before changing workflow logic.

⚠️ Don't ignore the 51,200 event / 50 MB ceiling. The Service warns at thresholds; act on it with Continue-As-New before the cap hits.

💡 **Takeaway:** Event History is the durable backbone — Commands become Events, Events drive replay, replay restores state. Keep workflow code deterministic and Continue-As-New before the History gets fat.
