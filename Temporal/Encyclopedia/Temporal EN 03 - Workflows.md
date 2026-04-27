# EN 03 — Workflows

🔑 **A Workflow is durable, replay-safe orchestration code that survives crashes, deployments, and time — written in a normal language but constrained to be deterministic.**

## Core Definitions

- **Workflow Definition** — *the code that defines your Workflow.* A function/class in Go, Java, Python, TS, .NET, Ruby, or PHP.
- **Workflow Type** — *the name that maps to a Workflow Definition.* Identifier that distinguishes one workflow type (e.g. `OrderProcessing`) from another (e.g. `CustomerOnboarding`).
- **Workflow Execution** — *a running Workflow, created by combining a Workflow Definition with a request to execute it.* The runtime instance.
- **Event History** — *a record of the Commands and Events that each Workflow Execution progresses through.* See [[Temporal EN 07 - Event History]].

## Identity & Chaining

- **Workflow ID** — caller-supplied business identifier; unique per Namespace among open executions.
- **Workflow Run ID** — Temporal-assigned UUID per attempt.
- **Workflow Run** — one link in the chain; identified by `(Workflow ID, Run ID)`.
- **Workflow Execution Chain** — *a sequence of Workflow Executions that share the same Workflow Id.* Links join via Continue-As-New, Retries, or Temporal Cron.

```
WF-ID = "order-42"

  Run A  ──Continue-As-New──▶  Run B  ──Retry──▶  Run C
 (run-id1)                    (run-id2)          (run-id3)
```

## Closed States

A Workflow Execution ends in one of:

| State | Meaning |
|---|---|
| Completed | Returned successfully. |
| Failed | Returned an error. |
| Cancelled | Successfully handled a cancellation request. |
| Terminated | Forcibly killed (no cleanup). |
| Continued-As-New | Restarted as a fresh run with new history. |
| Timed Out | Hit a workflow-level timeout. |

## Determinism

Replay re-executes the workflow code against the Event History. Commands generated during replay must match the recorded Events; mismatch → non-determinism error.

Required: *any time your Workflow code is executed it makes the same Workflow API calls in the same sequence, given the same input.*

**Forbidden in workflow code:**
- Wall-clock time (`time.Now()`) — use SDK's `workflow.Now`.
- Randomness — use SDK's deterministic RNG.
- Direct I/O (HTTP, DB, file) — push it into an Activity.
- Native threads / goroutines — use SDK concurrency APIs.
- Iterating maps with non-deterministic order.

**Operations that must not be reordered/added/removed without versioning:**
- Starting or cancelling Timers
- Scheduling Activity or Child Workflow executions
- Signalling external Workflows
- Scheduling Nexus operations
- Workflow completion/failure
- Patching or versioning calls
- Search Attributes and Memos modifications
- Side Effects

See [[Temporal RF 03 - Events]] for the underlying mechanics.

## Common Patterns

- **Activities** for any I/O or non-deterministic call. [[Temporal EN 04 - Activities]]
- **Child Workflows** to compose orchestrations. [[Temporal EN 09 - Child Workflows]]
- **Signals / Queries / Updates** for external interaction. [[Temporal EN 08 - Workflow Message Passing]]
- **Timers** for sleeps + scheduling.
- **Continue-As-New** when history grows too large (avoid the 50 MB / 51,200 event ceiling).
- **Schedules / Cron** to launch on a recurrence.

## Resilience

Workflows can *run — and keep running — for years, even if the underlying infrastructure fails*. The SDK automatically:
- Recreates pre-failure state via replay.
- Retries Activities per Retry Policy.
- Resumes Timers across worker death.
- Survives Service restarts and worker fleet rotation.

## Cross-Refs

- [[Temporal DV 02 - Workflow Basics]] — developer-side how-to
- [[Temporal CL 12 - workflow]] — CLI surface
- [[Temporal EN 05 - Detecting Application Failures]] — timeouts & retries
- [[Temporal EN 06 - Workers]] — what runs the code

⚠️ The number-one footgun: sneaking non-determinism into workflow code (a `Date.now()`, a `Math.random()`, a direct DB call). Replay will explode on the next worker restart, sometimes weeks later. Push *all* I/O and entropy through Activities or SDK helpers.

💡 **Takeaway:** workflows are deterministic orchestrators — durable, replayable, message-driven, and almost free to keep around when idle.
