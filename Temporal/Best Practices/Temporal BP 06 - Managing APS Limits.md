# BP 06 — Managing APS Limits

🔑 **Rate limiting in Temporal Cloud increases latency, not data loss — but design for your APS budget anyway, because latency *is* your customer's problem.**

## What Counts as an Action

> "Workflows: Starting, completing, resetting. Also starting Child Workflows, as well as Schedules and Timers. Activities: Starting, retrying, Heartbeating. Signals, Updates, and Queries."

**Rate limiting effect:**
> "In Temporal Cloud, the effect of rate limiting is increased latency, not lost work."

Track Action consumption in [[Temporal TC 11 - Billing and Usage]].

## Common Reasons You Hit APS Limits

### Bursty Traffic
- Implement application-level queuing or rate limiting to smooth predictable spikes.
- **Stagger** start times for scheduled batch operations — don't fire them simultaneously.
- Add **jitter** to high-volume Schedules and Workflow starts.
- Or accept the latency, or provision more capacity.

### Cascading Workflows / Fan-Out
- Question whether Child Workflows are necessary — Activities or separate Namespaces via Nexus may suffice.
- Limit fan-out by **batching items inside each Child Workflow** rather than one Child per item.
- Flatten deep hierarchies into shallower structures.

### Human-in-the-Loop at Scale
- Avoid polling — UIs constantly querying Workflow state burns Actions.
- Push state changes to a database for UI consumption.
- Minimize interaction-intensive queries on long-running Workflows.

### Real-Time SLAs / Deadline Management
- Use **longer monitoring intervals** (30 min, not 1 min).
- Consolidate multiple Timers into single checks.
- Have external systems **Signal** Workflows rather than relying on short-lived polling Timers.
- Apply **exponential backoff** with reasonable initial intervals.

### Many Small Activities
- Combine multiple external calls inside one Activity.
- Process datasets in chunks, not per-item.
- Batch operations where possible.

### Multiple Use-Cases in One Namespace
- Plan **separate Namespaces per use case** (one per environment).
- Don't concentrate different traffic patterns in a single namespace — see [[Temporal BP 05 - Managing Namespaces]].

## Capacity Modes

| Mode | Behavior |
|---|---|
| **On-Demand** (default) | Scales automatically based on trailing 7-day usage. Steady, predictable workloads. |
| **Provisioned** | Reserve capacity via **Temporal Resource Units (TRUs)** for guaranteed headroom during spikes. |

### When to Use Provisioned Capacity
- **Planned spikes** (promotions, launches): pre-provision before the event.
- **Unplanned spikes**: react instantly via UI / CLI / API once throttling is detected.
- **Load tests**: provision for the duration, deprovision after — see [[Temporal BP 03 - Pre-Production Testing]].
- **Batch jobs**: automate TRU scaling around schedules.
- **Migrations**: bridge the ~7-day gap while on-demand envelope adjusts.

### Cost Discipline
- Provision only when needed.
- **Deprovision promptly** after spikes end.
- Automate scaling for predictable patterns.

## Automation Best Practices

- Use **Cloud Ops API**, **Terraform Provider**, or **tcld** CLI for programmatic scaling.
- Set utilization thresholds — **scale up at 70–80% of limit**.
- Schedule capacity changes before known events using Temporal Schedules themselves.
- React to **leading indicators**: queue depth, campaign start times.

## Monitoring & Detection

- Track `temporal_cloud_v0_resource_exhausted_errors` for throttling events.
- Alert at **70–80% utilization** to provision proactively.
- Integrate **OpenMetrics** with your observability stack — see [[Temporal RF 01 - SDK Metrics]].
- Analyze historical patterns to choose reactive vs. proactive provisioning.

## Design Questions to Ask Before Coding

When designing new Workflows:

1. How many Actions does a single execution consume?
2. How many Workflows will run simultaneously?
3. What happens when **Actions × active Workflows scales 100×**?
4. Can operations be combined (bigger Activities, chunked data)?
5. Am I polling when **Signals** would do?
6. Must this Workflow run continuously, or can it be event-driven?

Run this checklist *before* shipping any new Workflow type into production.

## ⚠️ Anti-Patterns

⚠️ One namespace for every use case — they'll fight for the same APS pool.
⚠️ Polling Workflows for UI state — burns Actions and shifts cost to the user as latency.
⚠️ Fan-out: one Child Workflow per item in a 10,000-row dataset.
⚠️ Short polling Timers (1-min) for long deadlines (24-hour) — multiply across active Workflows.
⚠️ Triggering all scheduled batch jobs at exactly `00:00:00` UTC.
⚠️ Tight retry policies (sub-second, no backoff) hammering APS.
⚠️ Provisioning TRUs for a one-day spike and forgetting to deprovision.
⚠️ Heartbeating every second from a 5-second Activity.

## Cross-Refs

- [[Temporal EN 11 - Temporal Service]]
- [[Temporal BP 05 - Managing Namespaces]] — APS is per-namespace
- [[Temporal BP 09 - Cost Optimization]] — Actions = primary cost driver
- [[Temporal BP 03 - Pre-Production Testing]] — load-testing rate limits
- [[Temporal BP 02 - Worker Performance]]
- [[Temporal TC 11 - Billing and Usage]]
- [[Temporal RF 01 - SDK Metrics]]

💡 **Takeaway:** Smooth bursts with jitter + batching, prefer Signals over polling, alert at 70–80% APS, and treat TRU provisioning as a temporary tool, not a steady state.
