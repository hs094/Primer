# EN 02 — Temporal SDKs

🔑 **An SDK is the language-native bridge to the Temporal Service — Client API to drive workflows, Worker API to run them.**

## Definition

A Temporal SDK is an *open-source collection of tools, libraries, and APIs that enable Temporal Application development*. It abstracts distributed-systems plumbing (state machines, retries, replay, networking) so you write business logic in a regular language.

## Officially Supported Languages

Seven first-party SDKs:

- Go
- Java
- Python
- TypeScript
- .NET
- Ruby
- PHP

Third-party (community, not maintained by Temporal): Swift, Haskell, Clojure, Scala.

## What an SDK Provides

| Component | Responsibility |
|---|---|
| **Temporal Client** | Communication bridge — start workflows, signal/query/update, list executions, fetch results. |
| **Workflow API** | Define orchestration logic; call Activities; handle messages; invoke child workflows. |
| **Worker API** | Configure & run polling processes that execute Workflow + Activity code. |
| **Activity Execution API** | Customize timeouts, retry policies, heartbeats, async completion per call site. |
| **Data Converter** | Serialize payloads (default JSON) + optional encryption hooks. |
| **Interceptors** | Cross-cutting concerns (auth, tracing, metrics) around client + worker calls. |

## Client vs Worker

```
                  ┌──────────────────┐
                  │  Temporal Service │
                  └────▲───────▲─────┘
                       │       │
              start/   │       │  poll task queue
              signal/  │       │  return command/result
              query    │       │
                  ┌────┴───┐ ┌─┴─────────┐
                  │ Client │ │  Worker   │
                  │  API   │ │ Process   │
                  └────────┘ └───────────┘
```

- **Client** — anywhere your app talks to Temporal (HTTP handler, cron, CLI). Stateless w.r.t. workflows; addressable by Workflow ID. See [[Temporal CL 12 - workflow]].
- **Worker** — long-running process; loads Workflow + Activity registrations; long-polls a Task Queue. See [[Temporal CL 11 - worker]] and [[Temporal EN 06 - Workers]].

## Replay-Capable Workflow Code

The Service records each step of a Workflow Execution as an Event. On worker failure or rebalance, an SDK *replays* the workflow code against the Event History to reconstruct in-memory state, then resumes from the last recorded event. See [[Temporal EN 07 - Event History]].

This is why workflow code must be **deterministic**: the same input + same history must yield the same Command sequence. Random, wall-clock, I/O, and unmanaged concurrency are forbidden inside workflow code — the SDK supplies replay-safe substitutes (`workflow.Now`, `workflow.NewRandom`, etc.).

## What the SDK Replaces

Without Temporal you'd build:
- A retry/backoff library with persistent state
- A timer/scheduler service
- A state-machine framework with checkpointing
- A distributed lock + leader-election layer
- A "supervisor" that restarts work where it died

The SDK collapses all of those into ordinary function calls.

## Cross-Refs

- Concepts: [[Temporal DV 02 - Workflow Basics]], [[Temporal DV 10 - Activity Basics]], [[Temporal DV 15 - Worker Processes]]
- Messaging: [[Temporal EN 08 - Workflow Message Passing]], [[Temporal DV 07 - Message Passing]]
- Errors: [[Temporal RF 04 - Errors]], [[Temporal RF 05 - Failures]]

⚠️ Mixing SDK versions across replays of the same Workflow Execution can introduce non-determinism when SDK internals change. Use Worker Versioning / Patching when upgrading; don't just redeploy and hope.

💡 **Takeaway:** the SDK is the only sanctioned way to interact with Temporal — Client to start work, Worker to run it, both built around replay-safe abstractions.
