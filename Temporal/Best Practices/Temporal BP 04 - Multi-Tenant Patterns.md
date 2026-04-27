# BP 04 — Multi-Tenant Patterns

🔑 **Default to Task-Queue-per-tenant; only escalate to namespace-per-tenant for a small number of high-value tenants.**

## Four Core Patterns

### 1. Task Queues Per Tenant *(recommended)*
> "Use different Task Queues for each tenant's Workflows and Activities."

- Each Worker process polls multiple tenant Task Queues.
- Strong isolation, efficient resource utilization.
- Scales to **thousands of tenants per namespace**.
- Default starting point for most SaaS workloads.

### 2. Single Task Queue with Fairness
- Uses **fairness keys** to distribute work across tenants within one queue.
- "Priority and Fairness keys and weights can be adjusted without redeployment."
- Best when you have many tenants on different service tiers (free/pro/enterprise).

### 3. Shared Workflow TQ, Separate Activity TQs
- Workflow Tasks share one queue; Activity Tasks split per-tenant.
- Best for **lightweight Workflows + heavy, resource-intensive Activities** that need isolation.

### 4. Namespace Per Tenant
- Complete isolation — separate namespace per tenant.
- "Only practical for a smaller number of high-value tenants due to operational overhead."
- Manageable for **fewer than 50 tenants** without strong automation.
- See [[Temporal EN 12 - Namespaces]] and [[Temporal BP 05 - Managing Namespaces]].

## Picking a Pattern

| Tenants | Pattern |
|---|---|
| 1–50, high-value, strict compliance | Namespace per tenant |
| 50–thousands, mixed tiers | Task Queues per tenant |
| Many tenants, tier-based throttling | Single TQ + fairness |
| Mixed: light WF, heavy Activities | Shared WF TQ + per-tenant Activity TQs |

## Architectural Principles

- **Define the tenant model clearly** — what *is* a tenant? Org, user, workspace?
- **Prefer simplicity initially.** Start with fewer queues / namespaces; split when measured pain appears.
- **Understand deployment constraints** before locking in a pattern.
- **Test at scale before production** — see [[Temporal BP 03 - Pre-Production Testing]].
- **Plan for growth and onboarding** — how does adding tenant N+1 affect Workers, capacity, naming?

## Isolation Strategies

| Strategy | What it gives you |
|---|---|
| Task Queue isolation | Strong noisy-neighbor protection at queue level |
| Fairness-based isolation | Probabilistic, weight-based throttling |
| Activity-level isolation | Protects resource-heavy operations |
| Complete isolation | Separate namespaces — full data + compute boundary |

## Capacity Planning Levers

Workers manage:
- Concurrent Workflow / Activity execution sizes
- Poller counts
- Worker replicas

Namespace limits include:
- **Maximum Task Queue pollers: 20,000**
- Operations-per-second thresholds — must load-test, see [[Temporal BP 06 - Managing APS Limits]]

## Worked Example

A 1,000-customer SaaS provisioning **250 tenants per Worker**:

- 1,000 customers → **4 Workers minimum**
- 5,000 customers → **20+ Workers** (plus HA redundancy)

Always plan for HA replicas on top of base Worker count.

## Cross-Namespace Communication

When tenants live in separate namespaces, **use Temporal Nexus** to connect Workflows across namespace boundaries. Avoid building ad-hoc HTTP/gRPC bridges between namespaces — Nexus preserves Temporal's reliability semantics.

## Naming Inside a Shared Namespace

When teams or tenants share a namespace:
- Prefix Workflow IDs with tenant or service identifiers.
- Keep Task Queue names unique within the namespace.

```text
workflow_id = f"tenant-{tenant_id}-order-{order_id}"
task_queue  = f"orders-tenant-{tenant_id}"
```

## ⚠️ Anti-Patterns

⚠️ One namespace, one Task Queue, all tenants — guaranteed noisy-neighbor incident.
⚠️ Namespace-per-tenant at hundreds of tenants without automation — operational burnout.
⚠️ Skipping load testing before onboarding a large tenant.
⚠️ Hard-coding tenant IDs in Workflow code instead of passing them as inputs.
⚠️ Mixing dev / staging / prod tenants in one namespace.
⚠️ Building cross-namespace RPC by hand instead of using Nexus.

## Cross-Refs

- [[Temporal EN 12 - Namespaces]]
- [[Temporal BP 05 - Managing Namespaces]]
- [[Temporal BP 02 - Worker Performance]]
- [[Temporal BP 06 - Managing APS Limits]]
- [[Temporal BP 07 - Cloud Access Control]]
- [[Temporal BP 08 - Security Controls]]

💡 **Takeaway:** Start with Task-Queue-per-tenant; reach for Namespace-per-tenant only when isolation is contractual, not aesthetic.
