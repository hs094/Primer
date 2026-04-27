# BP 09 — Cost Optimization

🔑 **Actions are the dominant cost — optimize Activity granularity, retries, and Continue-As-New first; storage second; never at the expense of debuggability.**

## Cost Components

Temporal Cloud pricing has three pieces:

| Component | Share |
|---|---|
| **Actions** | Primary cost driver |
| Storage | Usually ≤ 10% of monthly bill |
| Support | Plan-dependent |

Track in [[Temporal TC 11 - Billing and Usage]].

## Optimization Priority

1. **Actions optimization** — biggest reduction.
2. **Active Storage** — relevant for long-running Workflows or large payloads.
3. **Retained Storage** — relevant for high volume + long retention.

## Actions Optimization

### Activity Granularity
Trade-off: more Activities = better observability and retry control, but **more Actions consumed**. Don't pre-emptively merge Activities — measure first.

### Child Workflows vs Activities
**Child Workflows cost 2 Actions** vs **Activities at 1 Action each.** Reach for Child Workflows only when you genuinely need separate Workflow lifecycle / Event History.

### Retry Policies
Each Activity retry = 1 Action. Tighten by:

- Setting `MaximumAttempts` to cap retries.
- Increasing `InitialInterval` to slow retry cadence.
- Adding error types to `NonRetryableErrorTypes` for unrecoverable cases.
- Using **next retry delay** for dynamic, failure-type-aware control.

### Local Activities
> "Multiple Local Activities that run back-to-back only count as a single billable action," but they lose independent retry control if any fail.

**Stick with Regular Activities when:**
- Activity takes 10+ seconds
- You need independent retry control
- You want to avoid re-running expensive Activities
- You need immediate Signal / Update handling
- You need separate resource management

### Batching
- **Search Attributes:** batch into a single `UpsertSearchAttributes` call (1 Action regardless of count).
- **Signals:** client-side dedup; use `SignalWithStart` instead of separate calls.

## Storage Optimization

### Active Storage (Open Workflows)

**Continue-As-New**
> "For long-running Workflows with extended sleep/wait periods, calling Continue-As-New before sleeping closes the current execution."

This is the single biggest active-storage win.

**Compression**
Custom Data Converters with compression help for moderately large payloads (100 KB – 1 MB).

**Claim Check Pattern**
Store large payloads externally (S3 / GCS); pass references through the Workflow. Same recommendation as in [[Temporal BP 02 - Worker Performance]] for Event-History health.

### Retained Storage (Closed Workflows)

- Default retention is **30 days** (configurable 1–90).
- Shorter retention reduces cost but limits historical analysis.
- **Workflow Export** to external storage for compliance while keeping retention short.
- Note: **export costs 1 Action per export**.

## Validation Framework

- **Test in non-production first.**
- **Monitor via Cloud UI Usage dashboard.**
- **Deploy progressively** — see Worker Versioning ([[Temporal BP 02 - Worker Performance]]).
- **Re-evaluate quarterly** as systems evolve.

### Success Criteria
- Cost reduction without **increased MTTR**.
- Workflow success rates maintained.
- No degraded incident detection (MTTD).

## Measurement Baseline

Establish *before* optimizing:

- Actions consumption per Workflow / day / month / Namespace
- Storage consumption (Active and Retained)
- Monthly costs by Namespace / Workflow Type
- Observability metrics — debugging time, incident detection

If you don't have this baseline, you can't tell whether your "optimization" actually saved money or just hid bugs.

## ⚠️ Anti-Patterns

⚠️ **Premature Activity consolidation** — combining Activities before you understand failure modes destroys observability and retry control.
⚠️ **Inappropriate Local Activities** — using them without understanding their lack of Worker-level isolation and retry semantics.
⚠️ **Missing Continue-As-New** — long-running Workflows accumulate large Event Histories, inflating storage cost.
⚠️ **High Activity retry volume** — usually a symptom of timeouts that are too short or flaky Activities, not "just the retry policy."
⚠️ **Large payloads through Workflows** — multi-MB blobs belong in S3 / blob storage, referenced by ID.
⚠️ **Over-optimization** — "aggressively optimizing costs without maintaining sufficient visibility for debugging and operational needs."
⚠️ **Excessive Heartbeats** — each is 1 Action; only heartbeat for Activities running 10+ minutes.
⚠️ Enabling HA on every namespace — **doubles consumption costs** ([[Temporal BP 08 - Security Controls]]).
⚠️ Optimizing without a baseline.

## Cross-Refs

- [[Temporal BP 06 - Managing APS Limits]] — Action-budgeting design questions
- [[Temporal BP 02 - Worker Performance]] — Event History hygiene + Versioning
- [[Temporal BP 05 - Managing Namespaces]] — per-namespace cost attribution
- [[Temporal BP 08 - Security Controls]] — HA cost doubling
- [[Temporal TC 11 - Billing and Usage]]
- [[Temporal RF 01 - SDK Metrics]]
- [[Temporal SH 05 - Production Checklist]]

💡 **Takeaway:** Cut Actions first (retries, fan-out, polling), then storage (Continue-As-New, Claim Check, retention), and never trade debuggability for a few percent — measure before and after, every time.
