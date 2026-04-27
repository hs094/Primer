# EN 01 — Temporal Platform

🔑 **A scalable, reliable runtime for durable function executions — code that runs exactly once, to completion, whether for seconds or years.**

## What Temporal Is

Temporal is a runtime for **durable function executions** called Workflow Executions. The platform guarantees applications execute reliably despite network outages, server crashes, or other infrastructure failures. Developers write business logic; Temporal handles failure recovery.

The platform separates concerns:
- **You** write Workflow + Activity code in a regular language (Go, Java, Python, TypeScript, .NET, Ruby, PHP).
- **Temporal** persists state, retries failures, schedules tasks, and replays history when something dies mid-flight.

## Durable Execution

State is preserved through an **Event History** — an append-only log of every step a Workflow Execution has taken. When a Worker crashes, a new Worker re-reads the history and rebuilds in-memory state by **replay** before continuing from the last recorded event. See [[Temporal EN 07 - Event History]].

Net effect: code runs *exactly once and to completion*, even across machine death, deployments, and multi-year sleeps.

## Platform Components

```
┌─────────────────────────────────────────────────────────┐
│                   Temporal Service                      │
│   (Go server + persistence DB; or Temporal Cloud SaaS)  │
│                                                         │
│   - History service   - Matching service                │
│   - Frontend          - Worker service                  │
└────────▲──────────────────────────▲─────────────────────┘
         │   gRPC (long poll)        │
         │                           │
   ┌─────┴──────┐             ┌──────┴──────┐
   │  Clients   │             │   Workers   │
   │ (start WF) │             │ (run code)  │
   └────────────┘             └─────────────┘
```

- **Temporal Service** — Go-written server backed by a database (Cassandra / MySQL / PostgreSQL). Orchestrates state, never runs your code. Managed variant: Temporal Cloud. See [[Temporal EN 11 - Temporal Service]].
- **Worker Processes** — *you* host these. They poll Task Queues, execute Workflow + Activity code, report results. See [[Temporal EN 06 - Workers]].
- **Workflow Executions** — lightweight reentrant processes with exclusive local state. Suspended ones consume ~no resources.

## Temporal Applications

A Temporal Application is millions–billions of Workflow Executions communicating via **message passing** (signals, queries, updates). Each execution is a **Reentrant Process**: resumable after suspension, recoverable after failure, reactive to external events. See [[Temporal EN 08 - Workflow Message Passing]].

## Core Primitives

| Primitive | Role |
|---|---|
| Workflow | Orchestrates work; deterministic; durable. [[Temporal EN 03 - Workflows]] |
| Activity | Single well-defined action; non-deterministic OK; retried. [[Temporal EN 04 - Activities]] |
| Worker | Polls Task Queue; runs code. [[Temporal EN 06 - Workers]] |
| Event History | Per-execution append-only log; basis for replay. [[Temporal EN 07 - Event History]] |
| Namespace | Isolation boundary inside a Service. [[Temporal EN 12 - Namespaces]] |
| SDK | Language client + worker library. [[Temporal EN 02 - Temporal SDKs]] |

## Failure Model

Temporal distinguishes:
- **Application-level failure** → surfaces as a Temporal Failure → workflow/activity treated as failed; retry policy applies. See [[Temporal RF 05 - Failures]].
- **Platform-level failure** (worker crash, network blip) → invisible to user code; replay restores state.

Timeouts detect application failure; retries mitigate it. See [[Temporal EN 05 - Detecting Application Failures]].

⚠️ The Temporal Service does **not** execute your code. If no Worker is polling a Task Queue, work just queues forever — silent stall. Always run a worker fleet for every Task Queue you start workflows on.

💡 **Takeaway:** Temporal = durable state machine + task scheduler. You write business logic as if nothing fails; the platform makes that assumption true.
