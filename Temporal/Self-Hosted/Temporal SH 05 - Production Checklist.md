# SH 05 — Production Checklist

🔑 **Self-hosted production is a database-class commitment: shard sizing is irreversible, you build the control plane, and missing observability shows up as silent backlog growth.**

## Scalability

- **Shard count is build-time.** `numHistoryShards` is set when the cluster is created and **cannot be adjusted later**. Resharding means a new cluster + replication. Size for 5–10 years of growth.
- Load-test before launch with **Omes** (Temporal's official benchmarking tool). Don't trust theoretical capacity.
- Scale rules of thumb:
  - History service scales with shard count and persistence throughput.
  - Frontend scales with client RPS.
  - Matching scales with task queue partitions.
  - Worker service scales with system workflow volume.

```yaml
persistence:
  numHistoryShards: 4096   # pick generously, irreversible
```

⚠️ Picking `numHistoryShards: 4` for a "small" cluster is the most common production-blocking decision. Default to **at least 512**, often 4096 for serious fleets.

## Availability

- Target **99.99%** uptime.
- Test **availability during upgrades** — see [[Temporal SH 10 - Upgrade Server]]. Rolling upgrades are required; full restarts are not acceptable.
- Run frontend / history / matching across multiple AZs.
- Persistence (Cassandra multi-DC, RDS multi-AZ) must match your availability target.

## Management & Control Plane (You Build It)

⚠️ **Self-hosted Temporal has no built-in RBAC, no audit logging, no namespace billing, no SSO control plane.** Either build them (custom `Authorizer` + log shipper + UI) or use [[Temporal TC 01 - Cloud Overview]].

See [[Temporal BP 04 - Multi-Tenant Patterns]], [[Temporal BP 08 - Security Controls]].

## Maintenance & Upgrades

- Stay on a **supported version**. Upgrade sequentially (`v1.n` → `v1.n+1`) — see [[Temporal SH 10 - Upgrade Server]].
- Schedule schema migrations during low-traffic windows.
- Test upgrades against a staging cluster with replayed production load before production rollout.
- Test workflow / activity worker shutdown during upgrade.

## Cost & Capacity

- DB IOPS dominates cost — provision generously and monitor saturation.
- Elasticsearch / OpenSearch for visibility is a separate bill; size shards and refresh intervals.
- Workers are usually the cheapest tier; over-provision them rather than starve.

## Observability — Three Metric Categories

See [[Temporal SH 08 - Monitoring]].

**Service metrics** (per-component health):
- `temporal_request_latency` per service (frontend / history / matching).
- `temporal_persistence_latency` — direct view into your DB.
- Frontend RPS, error rate, gRPC status code distribution.

**Persistence metrics:**
- Read/write latency p50/p99/p999.
- Connection pool saturation.
- Shard movement.

**Workflow execution metrics:**
- `temporal_sync_match_rate` — fraction of tasks dispatched without polling. **Target > 99%.** Lower means workers are starved.
- `temporal_schedule_to_start_latency` — time a task waited for a worker. High = add workers.
- Workflow / activity completion vs failure counts.
- Persistence task queue backlog.

## Required Pre-Launch Checklist

- [ ] `numHistoryShards` chosen for 5–10y growth, never reduced post-launch.
- [ ] mTLS on internode + frontend; clientCAs configured. See [[Temporal SH 07 - Security]].
- [ ] Custom `Authorizer` + `ClaimMapper` configured (default `noopAuthorizer` allows everything).
- [ ] Advanced visibility (ES/OpenSearch or PG12+/MySQL 8) wired and indexed. See [[Temporal SH 09 - Self-Hosted Visibility]].
- [ ] Metrics scraped to Prometheus / Datadog / equivalent. Dashboards imported.
- [ ] DB backup + PITR + tested restore.
- [ ] Schema migration runbook tested in staging.
- [ ] Namespace retention reviewed per environment. See [[Temporal SH 06 - Self-Hosted Namespaces]].
- [ ] Archival enabled if histories must outlive retention. See [[Temporal SH 11 - Archival and Replication]].
- [ ] Multi-cluster replication configured if DR is required.
- [ ] Worker auto-scaling on `schedule_to_start_latency` and CPU.
- [ ] Load test with Omes at 2x peak target.

## Expert-Review Items

- Worker tuning (concurrent activity / workflow task pollers).
- Workflow code review for non-determinism.
- Security review (TLS, claims, payload codec, audit log).
- DR runbook (failover via [[Temporal SH 11 - Archival and Replication]]).

⚠️ **Mission-critical Temporal requires real expense.** DB, observability, expert time. Budget realistically.

💡 **Takeaway:** The checklist's load-bearing items are shard count (irreversible), advanced visibility, mTLS + Authorizer, and observability into sync match rate / schedule-to-start. Skip any of these and you'll discover them under load.
