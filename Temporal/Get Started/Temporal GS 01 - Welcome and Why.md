# GS 01 — Welcome and Why

🔑 **Temporal makes distributed apps feel like local code: durable execution, visible state, less plumbing.**

## What Temporal is
A durable execution platform. You write business logic as ordinary code (a Workflow) and the platform guarantees it runs to completion — across crashes, restarts, deploys, network partitions, and timeouts that may stretch from seconds to years.

The pitch lands on three problems:

1. **Reliable distributed applications** — recover from process crashes and unreachable APIs without glue code.
2. **Productive code structure** — express business logic coherently instead of stitching queues, state machines, and cron jobs.
3. **Visible application state** — built-in CLI and Web UI for inspecting, debugging, and resolving live issues.

## Durable Execution
Once a Workflow Execution starts, it runs to completion. The [[Temporal EN 11 - Temporal Service]] persists every step to a database (the Event History), so on failure the [[Temporal EN 06 - Workers]] replay history and resume from the last checkpoint.

You write this:

```python
result = await workflow.execute_activity(
    charge_card,
    payment,
    schedule_to_close_timeout=timedelta(minutes=5),
)
```

You don't write: retry queues, dead-letter handlers, idempotency tables, cron-driven resumers, or saga-state databases. The platform owns failure-handling; your code owns intent.

## Why it beats rolling your own
Without Temporal, a long-running business process becomes:

- A queue (RabbitMQ / SQS) for the steps
- A state machine in a DB for "where are we"
- Cron or a scheduler to nudge stuck items
- Custom retry/backoff per integration
- A separate observability layer to see what's happening

Temporal collapses all of that into [[Temporal EN 03 - Workflows]] + [[Temporal EN 04 - Activities]] + a managed service. Failure handling, retries, timeouts, and history are platform concerns.

## Polyglot from day one
Open-source SDKs cover seven languages: .NET, Go, Java, PHP, Python, Ruby, TypeScript. Teams pick per-service; Workflows in different languages can call each other. Python users start at [[Temporal DV 01 - Python SDK Overview]].

## Visibility built in
- **Temporal CLI** — start, signal, query, terminate, inspect Workflows from the terminal. See [[Temporal CL 01 - CLI Overview]].
- **Web UI** — browser view of Workflow Executions, Event Histories, and stack traces.
- **Event History** — full, durable log of every event in a Workflow's lifetime. The same log that powers replay also powers debugging.

## Where it's running
Used by Stripe (payments), Coinbase (money movement), Netflix (CI/CD), Datadog (infra automation), Snap, Box, Maersk, Airbnb, plus AI-agent companies (Lindy, Dust). The pattern is the same across them: long-running, multi-step, must-not-lose-state work.

## Adoption paths
- **Self-host** — run the Temporal Service yourself; see [[Temporal GS 05 - Set Up Local Python]] for the local dev loop.
- **Temporal Cloud** — managed service, same SDKs. See [[Temporal TC 01 - Cloud Overview]].

⚠️ Don't reach for Temporal for sub-millisecond hot paths or simple stateless request/response — the value shows up when work spans many steps, services, or time. For pure RPC, just write a service.

💡 **Takeaway:** Temporal turns the question "how do I make this distributed workflow reliable?" into "what should the workflow do?" — the platform owns the rest.
