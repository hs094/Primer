# EN 06 — Workers

🔑 **A Worker is your process polling a Task Queue and running your code — the Temporal Service never executes user code itself.**

## Three Things Called "Worker"

The term refers to three related-but-distinct concepts:

- **Worker Program** — *the static code that defines the constraints of the Worker Process, developed using the APIs of a Temporal SDK.* The source you write.
- **Worker Process** — *responsible for polling a Task Queue, dequeueing a Task, executing your code in response to a Task, and responding to the Temporal Service with the results.* The OS process running on your infra.
- **Worker Entity** — *the individual Worker within a Worker Process that listens to a specific Task Queue.* Contains a Workflow Worker and/or Activity Worker that drive Workflow and Activity Executions.

A single Worker Process can host multiple Worker Entities, each on a different Task Queue.

## Architecture

```
┌────────────────────────────── Temporal Service ──────────────────────────────┐
│       Task Queue: orders                Task Queue: emails                   │
│       ┌─ WF tasks ─────┐                ┌─ Activity tasks ─┐                 │
│       │ ▢ ▢ ▢ ▢ ▢      │                │ ▢ ▢ ▢            │                 │
│       └────────────────┘                └──────────────────┘                 │
└──────────▲──────▲──────────────────────────────▲────────────────────────────┘
           │ poll │                              │ poll
           │      │                              │
       ┌───┴──┐ ┌─┴────┐                     ┌──┴────┐
       │ WP-1 │ │ WP-2 │                     │ EmailWP│
       └──────┘ └──────┘                     └────────┘
       Worker Processes (your hosts/pods)
```

Worker Processes long-poll Task Queues, execute the work, post results back. The Service is a *choreographer* — it orchestrates state transitions and dispatches tasks; it does not run user code.

## Tasks

- **Workflow Task** — unit of work for a Workflow Execution. Contains the next batch of history events; the Worker advances the workflow code against them and emits Commands.
- **Activity Task** — unit of work for an Activity Execution. Worker invokes the registered Activity Function and reports completion.

A Worker Entity has a Workflow Worker, an Activity Worker, or both — based on what was registered.

## Worker Identity

*The individual Worker identifier that helps identify the specific Worker instance.* Default format: `${process.pid}@${os.hostname()}`.

In cloud environments, customize to include ECS task IDs, Kubernetes pod names, deployment env, region, etc. — meaningful identity matters for debugging stuck workflows in [[Temporal EN 10 - Visibility]].

## Statelessness & Resumption

Worker Entities are **stateless** w.r.t. workflow state — all durable state lives in the Service's Event History. A single worker can manage millions of *open* Workflow Executions because suspended ones consume nothing on the worker; the Service hands out tasks only when there's work.

When a worker dies mid-Workflow-Task, the Service reassigns the task to another Worker, which **replays** the Event History to rebuild in-memory state, then continues. See [[Temporal EN 07 - Event History]].

## Sticky Execution

After a Worker successfully completes a Workflow Task, the Service prefers routing the *next* task for that Workflow Execution to the *same* Worker — its in-memory state is already hot, so it can skip a full replay. Falls back to any worker if the sticky one is unreachable.

## Worker Versioning

Lets you pin a Workflow Execution to a specific code version so a rolling deploy doesn't introduce non-determinism mid-flight. New executions go to the latest version; old ones keep replaying against the version they started under.

## Polling Mechanics

- Long-polls (~60s) reduce request volume.
- Worker pulls *only* when it has slots free (concurrency limits per Activity/Workflow Worker).
- No backpressure config on the Service side — the Worker controls its own pull rate.

## Deployment Patterns

- **Per-domain** Task Queue (`orders`, `emails`, `billing`) → independent scaling and blast radius.
- **Worker fleet** of N replicas per Task Queue → horizontal scale.
- **Co-locate** Workflow + Activity Workers in one process for simple apps; split for hot-path Activities (GPU, big files).
- **Resource-aware** routing via dedicated Task Queues for heavy work.

## Cross-Refs

- [[Temporal CL 11 - worker]] — CLI for inspecting workers
- [[Temporal DV 15 - Worker Processes]] — developer guide
- [[Temporal EN 03 - Workflows]] — what runs on Workflow Workers
- [[Temporal EN 04 - Activities]] — what runs on Activity Workers
- [[Temporal EN 07 - Event History]] — replay mechanism

⚠️ Workflow Code changes deployed to running workers can break replay if they alter the Command sequence (different activity, reordered call, removed branch). Use Worker Versioning or Patching/Versioning APIs — don't just push.

⚠️ A workflow with no worker polling its Task Queue stalls silently. Monitor Task Queue backlog, not just worker CPU.

💡 **Takeaway:** workers are dumb pollers running deterministic code; the Service holds all the state. Scale workers horizontally per Task Queue, version them when changing workflow logic.
