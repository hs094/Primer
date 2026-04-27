# GS 04 — Use Cases and Design Patterns

🔑 **If the work is multi-step, long-running, must-not-lose-state, or human-in-the-loop, Temporal is the shape that fits.**

## Core design patterns
The docs name two explicitly; both are textbook distributed-systems patterns that Temporal makes ergonomic.

### Saga
> A design pattern used to manage and handle failures in complex Workflows by breaking down a transaction into a series of smaller, manageable sub-transactions.

Each step has a compensating action. On failure, the Workflow walks the compensation chain in reverse. With Temporal you express it as plain `try/except` around `execute_activity` calls — no separate state machine.

```python
@workflow.defn
class BookTrip:
    @workflow.run
    async def run(self, req):
        compensations = []
        try:
            await workflow.execute_activity(book_flight, req)
            compensations.append(cancel_flight)
            await workflow.execute_activity(book_hotel, req)
            compensations.append(cancel_hotel)
            await workflow.execute_activity(charge_card, req)
        except Exception:
            for comp in reversed(compensations):
                await workflow.execute_activity(comp, req)
            raise
```

### State Machine
> A software design pattern used to modify a system's behavior in response to changes in its state.

A long-lived Workflow *is* a state machine: the Workflow function holds state in local variables; signals drive transitions; the Event History is the audit log. You don't write the table of `(state, event) → next_state`; you write Python control flow.

## Production use cases
The same primitives recur across very different industries.

**Transactions / money movement**
- **Stripe** — payment processing
- **Coinbase** — money movement
- **Box** — content management

**Business processes**
- **Turo** — bookings
- **Maersk** — orders / logistics
- **Airbnb** — marketing campaigns
- **Checkr** — human-in-the-loop background checks

**Entity lifecycle**
- **ANZ** — mortgage underwriting applications (long-running, multi-actor)
- **Yum! Brands** — menu versioning across stores

**Operations / infrastructure**
- **Datadog** — infrastructure services automation
- **Netflix** — custom CI/CD orchestration

**AI / ML and data engineering**
- **Descript** — video processing orchestration
- **Neosync** — data pipeline automation

**AI agents**
- **Lindy** — reliable, observable agents
- **Dust** — long-running, durable agents
- **ZoomInfo** — account summary generation

## General-purpose patterns
The docs call out three cross-cutting categories:

### Human in the Loop
Workflows wait for human input — approvals, form submissions, manual reviews — for arbitrary durations. Implementation: a Workflow blocks on a Signal (or on an Update), often guarded by a timeout that escalates if no response arrives.

### Polyglot Systems
Different services in different SDKs (Go + Python + Java) coordinate through the same Temporal Service. Workflows in one language call Activities or child Workflows in another.

### Long Running Tasks
Activities heartbeat progress so the Service knows they're alive. On Worker failure, another Worker picks up the same Activity from its last checkpoint instead of restarting from zero.

## When Temporal fits
Pattern-matchers:

- Multi-step process where each step can fail independently
- Process that lives longer than a single request — minutes, days, months
- Requires retry / timeout logic per step
- Needs an audit trail of "what happened"
- Mixes automated and human steps
- Coordinates multiple services or external APIs

## When it doesn't
- Pure synchronous request/response with no orchestration — just write the endpoint.
- Hot-path / sub-millisecond latency — Temporal adds overhead per Workflow Task.
- Single-step Activity — no orchestration value to add.

⚠️ "We have an AI agent" is not by itself a Temporal use case. It becomes one when the agent has a multi-step plan, makes external tool calls that can fail, runs longer than a request, or needs human approval mid-flight — exactly the durable-execution criteria.

💡 **Takeaway:** Saga + State-Machine cover most needs; the use-case list is just "anywhere reliability beats latency."
