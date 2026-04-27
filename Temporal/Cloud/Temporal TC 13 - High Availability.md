# TC 13 — High Availability

🔑 **Standard namespaces already span 3 AZs; HA adds asynchronous cross-region (or cross-cloud) replication for a 99.99% SLA, ~1-minute RPO, and ~20-minute RTO on the bigger failure modes.**

## What you get out of the box
Every Cloud namespace replicates synchronously across **three Availability Zones** in its primary region. Writes are durable before the service acks the client. Standard namespaces carry a **99.9% SLA** and absorb AZ failures with **zero RPO and near-zero RTO** automatically.

That's the floor. HA is an upgrade on top.

## High Availability namespaces
Enabling HA flips the namespace to **asynchronous replication across multiple isolation domains** and bumps the SLA to **99.99%**. Two deployment shapes:

- **Multi-region replication** — primary and replica in two regions on the same continent. Protects against regional outage; keeps data on one cloud and one continent.
- **Multi-cloud replication** — primary and replica on different cloud providers. Protects against provider-wide events; data crosses encrypted between clouds.

⚠️ Replica regions must be on the **same continent** as the primary, which excludes some Cloud regions from HA today.

## RPO and RTO targets
Cloud publishes recovery objectives by failure scope:

| Failure scope | Applies to | RPO | RTO |
|---|---|---|---|
| Availability Zone outage | All namespaces | Zero | Near-zero |
| Cell outage | HA-enabled (any flavor) | 1 minute | 20 minutes |
| Regional outage | Multi-region or Multi-cloud | 1 minute | 20 minutes |
| Cloud-wide outage | Multi-cloud only | 1 minute | 20 minutes |

These objectives apply **only to namespaces with Temporal-initiated failovers enabled** — the default. Temporal strongly recommends keeping that on.

All namespaces are also backed up every **4 hours** for catastrophic recovery.

## How Cloud meets the numbers
- **Asynchronous replication** — replica catches up without slowing the active region.
- **Automatic outage detection** — extensive internal alerting, 24/7 on-call.
- **Battle-tested failover workflows** — same playbooks executed in regular DR drills against internal namespaces.
- **Conflict resolution** — when failover happens before the replica is fully caught up, Cloud reconciles divergent histories.
- **Transparent DNS routing** — your Workers don't change endpoints; DNS forwards traffic to the new active region during failover.

## Workers and Workflows during failover
- Workers and Workflows see no performance impact from replication — it's out-of-band.
- During failover, in-flight tasks may need to be retried by the SDK; long-running Workflows resume from the last replicated event.
- Workers are configured with a single namespace endpoint and pick up the new active region via DNS without restart.
- Pin Workers to a region only when you have data-residency reasons; otherwise let DNS do its job.

## Failover modes
- **Automatic** — Temporal initiates failover when health probes trip. Default for HA namespaces.
- **Manual** — operators trigger a failover for testing or ahead of a planned regional change.

Replication lag is exposed as a metric so you can monitor in real time — combine with [[Temporal TC 10 - Cloud Metrics]] alerts.

## Costs and trade-offs
HA roughly doubles the data footprint and adds Action accounting on the replica side. Budget accordingly via [[Temporal TC 11 - Billing and Usage]] and [[Temporal BP 09 - Cost Optimization]]. The math is generally a no-brainer for revenue-critical workloads and overkill for low-stakes batch.

⚠️ Async replication means a regional or cloud-wide outage can drop **up to ~1 minute** of recent events. Workflows that must not lose any state should still use Activity-level idempotency and Signal acknowledgement — replication is not "exactly-once across the planet."

## Operational checklist
- Decide multi-region vs multi-cloud based on threat model, not vendor mood.
- Keep Temporal-initiated failovers **enabled**; don't trade your published RTO for control you won't actually exercise at 3am.
- Drill manual failover at least quarterly so your runbook isn't theoretical.
- Alert on replication lag exceeding your RPO budget.
- Track failover events in audit logs (see [[Temporal TC 14 - Cloud Operations and tcld]]).
- Combine with [[Temporal SH 08 - Monitoring]] dashboards so app-side and platform-side alerts share an SRE workflow.

## When HA is not enough
HA gives you availability and recovery, not data residency. If a workload must stay in a single jurisdiction, multi-region within that jurisdiction is the only HA option — cross-jurisdiction replication isn't supported. Coordinate with [[Temporal EN 12 - Namespaces]] design and [[Temporal BP 08 - Security Controls]] before assuming HA solves a compliance question.

## Comparing the deployment shapes

| Shape | Failure scope covered | Data crosses | When to pick |
|---|---|---|---|
| Standard (3 AZ) | AZ | within region | default; non-critical or cost-sensitive |
| Multi-region HA | Cell, regional | between regions, same cloud | most production workloads |
| Multi-cloud HA | Cell, regional, cloud-wide | between clouds (encrypted) | regulator/customer requires provider diversity |

The right choice usually falls out of the threat model, not the spec sheet. If you can't articulate what fails when, default to standard until you can.

## Configuration and tooling
- Enable HA on a namespace via the UI, [[Temporal TC 14 - Cloud Operations and tcld]] (`tcld namespace ...`), or the Terraform provider.
- HA is namespace-scoped — mix and match per workload inside the same Cloud account.
- Replication lag, failover events, and replica health show up in [[Temporal TC 10 - Cloud Metrics]]; failover audit entries land in audit logs.
- Combine with [[Temporal SH 08 - Monitoring]] dashboards so app-side and platform-side alerts share an SRE workflow.

## Testing failover
Manual failovers exist precisely so you can drill them:

1. Schedule a window; warn dependents.
2. Trigger via UI or `tcld`.
3. Watch Workers reconnect through DNS — no restart should be required.
4. Confirm new Workflow Executions complete normally; verify replica writes lag-free.
5. Fail back when satisfied.

A drill that reveals a broken runbook is cheap; an outage that reveals one is expensive.

## Worker placement notes
- Default: one fleet per region/zone with the same namespace endpoint configured. DNS handles the rest.
- For lower latency, you can deploy Workers near both the primary and replica regions; Cloud will route them transparently.
- Pin Workers to a region only when data residency or a hard latency budget demands it.

💡 **Takeaway:** Standard namespaces handle AZ failure for free; HA is the answer for region- and cloud-level events. The published numbers — 99.99%, 1m RPO, 20m RTO — depend on you keeping Temporal-initiated failovers on, drilling manual failovers, and budgeting for the doubled footprint.
