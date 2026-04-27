# EN 08 — Workflow Message Passing

🔑 **Workflows are stateful services: Queries read state, Signals fire-and-forget writes, Updates are synchronous validated writes.**

## Three Message Types

Temporal exposes three primitives for talking to a running Workflow Execution:

- **Query** — read-only request that returns workflow state without writing to history.
- **Signal** — asynchronous write; sender does not await a result.
- **Update** — synchronous write; sender awaits acceptance and/or completion.

All three are language-agnostic concepts implemented across the SDKs (Go, Java, Python, TypeScript, .NET, PHP) and surfaced via the CLI.

## Queries

A Query is a synchronous, read-only call into a workflow. Because it never produces a history event, queries are cheap and can run against **completed** workflows. They are ideal for inspecting current state.

The SDKs ship a built-in `__stack_trace` query that returns the workflow's current call stack — useful for live debugging of running executions.

⚠️ Queries must be deterministic and side-effect free. Mutating workflow state inside a query handler corrupts replay.

## Signals

Signals are asynchronous, fire-and-forget messages. The sender knows the signal was delivered to the service, but does not learn whether (or when) the handler ran or whether it succeeded.

Key properties:
- No result, no exception propagation back to caller.
- Worker need **not** be available at send time — signals are persisted and delivered when a worker is ready.
- Unlimited concurrent signals per workflow.
- **Signal-with-Start**: if the target workflow does not exist, it is started and the signal is delivered atomically.

Use signals when the caller cares only about delivery, not outcome.

## Updates

Updates are synchronous write operations. The caller awaits one of two stages:

- **Accepted** — the worker received the update and any validator passed.
- **Completed** — the handler finished and a result (or failure) is returned.

Updates require a worker to be available to validate and acknowledge. They have per-workflow concurrency caps and a per-run total cap, enforced on both Cloud and self-hosted.

**Update-with-Start** combines workflow start and update into one roundtrip — used in latency-sensitive flows like cart checkout or payment capture. It requires a `WorkflowIDConflictPolicy`.

⚠️ Update-with-Start is **not atomic**. If the update cannot be delivered, the workflow still starts. Plan for the orphan-start case.

## Signals vs. Updates

| Property | Signal | Update |
|---|---|---|
| Result to caller | No | Yes (accept + complete) |
| Worker required at send | No | Yes |
| Validator support | No | Yes (rejects before persist) |
| Concurrency per workflow | Unlimited | Bounded |
| Latency | Not in critical path | In critical path |

Pick signals when you want fan-in or eventual consistency. Pick updates when the caller must know the result.

## Queries vs. Updates for Reads

Queries are cheaper (no history event) and work on closed workflows. But if the read must wait for a state condition, an update with a wait-condition is more efficient than client-side polling with queries.

## Validators

Updates support a separate validator function that can reject the request **before** it is written to history. A rejected update produces no event — useful for input validation against current state.

⚠️ Validators must be deterministic and read-only, exactly like query handlers. Replay applies.

## Handler Patterns

Common patterns:
- One handler per message type, registered at workflow start.
- Handlers can `await` on workflow state (`Workflow.await(condition)`) before acting.
- On workflow completion, the SDK can be told to wait for in-flight handlers to drain — otherwise pending updates may be lost.

⚠️ A workflow that returns while update or signal handlers are still running risks dropping their work. Drain handlers explicitly before returning.

## Cross-references

- Developer guide: [[Temporal DV 07 - Message Passing]]
- CLI surface: [[Temporal CL 12 - workflow]]
- Related encyclopedia: [[Temporal EN 03 - Workflows]], [[Temporal EN 06 - Workers]]
- Visibility for discovery: [[Temporal EN 10 - Visibility]]

💡 **Takeaway:** Query for cheap reads, Signal for async writes you don't need to confirm, Update for writes the caller must validate or await.
