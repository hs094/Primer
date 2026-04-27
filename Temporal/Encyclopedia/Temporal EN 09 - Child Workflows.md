# EN 09 — Child Workflows

🔑 **A Child Workflow is a workflow spawned by another workflow in the same Namespace — full workflow semantics, but billed against the parent's history.**

## Definition

A Child Workflow Execution is a Workflow Execution started from inside another (parent) Workflow Execution. The parent must await the spawn (so the child's existence is durable in history) but may optionally await the result.

Both parent and child live in the same Namespace. A workflow can be both a parent and a child — nesting is allowed.

## Parent / Child Lifecycle

- **Spawn** is awaited; once the child is started, the parent has a durable handle.
- **Result** is optional — the parent can fire-and-forget or await completion.
- **Continue-As-New on the parent does not carry children over.** Children continue running but the new parent run does not own them.
- **Continue-As-New on the child** appears as a single execution from the parent's perspective.
- When the parent closes (completes, fails, or is cancelled), each child is reconciled per its **Parent Close Policy**.

## Parent Close Policies

Three policies govern what happens to children when the parent closes:

- **Terminate** (default) — children are terminated.
- **Cancel** — children receive cancellation; they choose how to react.
- **Abandon** — children outlive the parent and run independently.

⚠️ The default is Terminate. If you want a child to keep running after the parent finishes, you must explicitly set Abandon.

## When to Use Child Workflows

Child workflows make sense when:

1. **Separate service / different worker pool.** Child runs on its own task queue with independent state and async communication.
2. **Workload partitioning.** A parent can spawn ~1,000 children, each driving ~1,000 activities — fanning out to ~1M activities while keeping each event history bounded.
3. **One-to-one with a resource.** E.g. one child workflow per host being upgraded.
4. **Periodic / long-running logic via Continue-As-New** without bloating the parent's event history.

## Child Workflow vs. Activity

| | Child Workflow | Activity |
|---|---|---|
| API surface | Full workflow APIs (timers, signals, queries, child spawn) | Limited; no workflow APIs |
| Determinism | Required | Not required |
| Cancellation w/ parent | Configurable via Parent Close Policy | Always cancelled |
| History footprint | Own event history | Parent records input/output/retries |
| Cost per call | Several events | Few events |

**Rule of thumb: when in doubt, use an Activity.** Reach for a child workflow when you genuinely need workflow-only capabilities (timers, signals, child spawn, durable state machine).

## Event History Limits

A workflow has a hard limit on event history size and count. Each child spawn, signal, and result adds events to the parent. As a guideline:

- A single parent should not spawn more than ~1,000 children directly.
- For larger fan-out, build a tree (parent → children → grandchildren).

⚠️ Over-using child workflows for what an activity could do bloats event history and slows replay. Activities are cheaper per call.

## State Sharing

Parent and child do **not** share local state. All communication goes through:
- Workflow input / return value
- Signals / updates / queries
- Shared external state (database, cache) accessed via activities

## Cross-references

- Developer guide: [[Temporal DV 03 - Child Workflows]]
- Message passing across the boundary: [[Temporal EN 08 - Workflow Message Passing]]
- Continue-As-New mechanics: [[Temporal EN 03 - Workflows]]
- Worker / task queue isolation: [[Temporal EN 06 - Workers]]

💡 **Takeaway:** Child workflows give you full workflow semantics for sub-tasks — use them for partitioning, isolation, or periodic loops, but prefer activities for plain external calls.
