# BP 03 — Pre-Production Testing

🔑 **"Failure is normal." Temporal survives failures; only relentless pre-prod testing proves your application does too.**

## Guiding Principles

1. **Failure is normal.** Temporal is designed to survive failure — your application logic must be too.
2. **Partial failure is the worst case.** Systems "mostly working" expose the most flaws.
3. **Recovery paths deserve as much testing as steady-state.**
4. **Observability first.** Metrics, logs, and visibility tools must exist *before* injecting failures.
5. **Continuous process.** Testing is never finished.

## Worker Chaos Drills

### Kill All Workers Then Restart
- Abruptly terminate every Worker on a Task Queue, then restart.
- Validates at-least-once execution semantics.
- Verifies Activities are idempotent and Workflows replay cleanly.
- Watch for: duplicate Activity results, Workflow failures, backlog growth.

### Frequent Worker Restarts
- Restart 20–30% of the fleet every few minutes.
- Mimics OOM and CPU-saturation failures.
- Validates Sticky Task Queue invalidation and rescheduling.
- Watch: replay latency, completion rates, Activity result correctness.

## Load Testing

### Pre-conditions for a Useful Load Test
1. SDK metrics accessible — not just Cloud metrics. ([[Temporal RF 01 - SDK Metrics]])
2. Predict expected behavior: rate-limit thresholds, Workflow failure modes, latency curves, Worker metrics.
3. Determine throughput requirements and **coordinate capacity with the account team** ahead of time.
4. Automate startup/shutdown; define explicit success criteria.

### Validate Downstream Capacity
Schedule Workflows that **deliberately exceed** downstream throughput limits:
- Monitor downstream error rates, latency, saturation.
- Track Activity failure classification and retry behavior.
- Verify data correctness and Activity idempotency under stress.

### Validate Rate Limiting
> "In Temporal Cloud, the effect of rate limiting is increased latency, not lost work."

- Measure current APS at target throughput.
- Observe Worker and client behavior when throttled.
- Track `temporal_request_failure` and end-to-end Workflow latency.

See [[Temporal BP 06 - Managing APS Limits]].

## Failover & Availability Testing

### Region Failover
- Manually fail over a Namespace using High Availability features.
- Real outages are messy and rarely isolated — practice the messy case.
- Monitor Namespace availability, client/Worker connectivity, Workflow Task reassignments.

## Dependency / Downstream Testing

Intentionally degrade what your Activities depend on:
- Make databases read-only or unavailable.
- Inject high latency or error rates into external APIs.
- Throttle or pause message queues.

Track Activity retry behavior, heartbeat effectiveness, connection pool exhaustion, failure propagation.

## Deployment & Code Testing

### Deploy with a Versioning Strategy
- Use Worker Versioning to avoid non-determinism errors (NDEs).
- Validate Workflow success and backlog drain.
- Monitor Workflow Task failure reasons.

### Recover from NDEs Deliberately
- Deploy code that intentionally causes an NDE.
- Apply rollback / versioning to recover.
- Drain or reset the task backlog.
- This tests *operator discipline*, not just code.

## Network-Level Testing

### Cut Network Connectivity
Temporarily block all traffic between Workers and the Temporal service. Tools: Kubernetes NetworkPolicy, ToxiProxy, Chaos Mesh, local firewall rules.

Validates: Worker retry, Sticky Task Queue handling, Workflow Task timeouts, Activity retry semantics, replay correctness.

## Observability Checklist (during tests)

- Workflow Task & Activity failure rates
- Throughput and limits used
- End-to-end latency
- Task latency and backlog depth
- Workflow History size
- Worker CPU, memory, restart count
- gRPC error codes
- Retry behavior

## Game Day Runbook

### Before
- Notify API teams and Temporal Cloud Support.
- Display SDK + Cloud metric dashboards.
- Mute or reroute alerts.
- Verify rollback and scale controls.

### During
- One variable at a time.
- Record start/stop per experiment.
- Capture screenshots/logs of unexpected behavior.
- Track backlog growth and drain rate.

### Recovery Validation
- Workflows resume with no manual intervention.
- No permanent Workflow Task failures (unless intentional).
- Activity retries behave as expected.
- Backlogs drain in predictable time.

### After-Action Review
- Identify unclear alerts and missing metrics.
- Update retry / timeout / versioning policies.
- Document surprises and operational debt.

## ⚠️ Anti-Patterns

⚠️ Non-idempotent Activities (chaos testing exposes them mercilessly).
⚠️ Infinite retries with no circuit breaker.
⚠️ Using Workflow logic to "wait out" broken dependencies.
⚠️ Load testing without SDK metrics — you'll be flying blind.
⚠️ Running multiple chaos experiments simultaneously.
⚠️ Skipping the after-action review.

## Cross-Refs

- [[Temporal DV 19 - Testing]] — unit/integration test framework
- [[Temporal BP 02 - Worker Performance]] — what to scale during recovery
- [[Temporal BP 06 - Managing APS Limits]] — provisioning before load tests
- [[Temporal SH 05 - Production Checklist]]
- [[Temporal EN 11 - Temporal Service]]

💡 **Takeaway:** Game-day your Workers, network, dependencies, and deploys **before** customers do — operability under stress is the only real production-readiness signal.
