# EN 13 — Temporal Nexus

🔑 **Nexus is Temporal's cross-namespace RPC layer: teams expose reusable Operations through endpoints without coupling to each other's internals.**

## What Nexus Solves

Inside a single Namespace, workflows talk freely. Across Namespaces (different teams, services, or trust boundaries) you historically had to expose your own RPC, manage retries, and reinvent durable cross-service coordination.

**Temporal Nexus** is a peer-to-peer system that lets one application invoke functionality in another Namespace through a reverse-proxy endpoint, with Temporal's reliability primitives baked in. As of GA it ships on both Temporal Cloud and self-hosted deployments.

## Core Components

### Service

A **Nexus Service** is a named collection of related **Operations**. Callers depend on the service contract; the handler chooses how to implement each operation (start a workflow, send a signal, run plain code).

### Operation

An Operation is a single named, typed call. There are two execution modes:

- **Asynchronous** — starts a Workflow on the handler side, optionally with Eager Start. The operation can run for up to **60 days**, mirroring long-running workflow semantics.
- **Synchronous** — must complete within a **10-second deadline**. Suitable for Signals, Queries, Updates, or low-latency external calls.

### Endpoint

An **Endpoint** is the reverse-proxy address callers use. It decouples the caller from the handler's location: the endpoint maps to a target Namespace + Task Queue.

### Registry

The **Nexus Registry** (UI, CLI, or Cloud Ops API) manages endpoint definitions — which endpoint resolves to which target Namespace + Task Queue.

## How a Call Flows

1. Caller workflow (in Namespace A) invokes an Operation on an Endpoint.
2. The endpoint resolves to a target Namespace B + Task Queue.
3. A Handler Worker in Namespace B polls that Nexus Task Queue, just like a normal task queue.
4. The handler runs (sync) or starts a workflow (async) and returns a result or handle.
5. Caller's workflow durably awaits the result.

## Built-in Reliability

Nexus reuses Temporal's queue-based worker model, so the caller doesn't implement retries:

- **Automatic retries** with exponential backoff.
- **Rate limiting and concurrency limits** at the endpoint.
- **Circuit breaking** after 5 consecutive retryable errors.
- **At-least-once execution** semantics via the Nexus RPC protocol.
- If the handler service is down, callers continue **scheduling** Operations; processing resumes when the handler comes back.

## Multi-level Composition

Operations compose across services, end-to-end:

```
Workflow A (Namespace 1)
  → Nexus Operation 1
    → Workflow B (Namespace 2)
      → Nexus Operation 2
        → Workflow C (Namespace 3)
```

Each hop is durable — failures in any segment retry independently without losing the parent's progress.

## When to Use Nexus

- Cross-team / cross-namespace integration where you want a stable contract.
- Replacing ad-hoc HTTP between Temporal applications.
- Long-running async work in another team's namespace (async Operations up to 60 days).
- Synchronous low-latency lookups against another team's workflow (sync Operations under 10s).

## When Not to Use Nexus

⚠️ Nexus is for **inter-namespace** coordination. Inside a single namespace, just call workflows / activities / child workflows directly — adding Nexus is unnecessary indirection.

⚠️ Synchronous Operations have a hard 10-second deadline. If the handler can't reliably finish in that window, model it as an async Operation backed by a workflow.

## Operator Surface

Nexus endpoints are managed via:
- The Temporal UI.
- The CLI (`temporal operator nexus ...`) — see [[Temporal CL 07 - operator]].
- The Cloud Ops API (Cloud only).

## Cross-references

- Namespace boundaries Nexus crosses: [[Temporal EN 12 - Namespaces]]
- Underlying service that hosts endpoints: [[Temporal EN 11 - Temporal Service]]
- Cloud namespace setup: [[Temporal TC 03 - Cloud Namespaces]], [[Temporal TC 01 - Cloud Overview]]
- Self-hosted namespace setup: [[Temporal SH 06 - Self-Hosted Namespaces]]

💡 **Takeaway:** Nexus = durable, retried, rate-limited cross-namespace RPC. Sync ops for sub-10s reads, async ops for workflow-backed work up to 60 days.
