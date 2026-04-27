# SH 10 — Upgrade Server

🔑 **Upgrade sequentially: `v1.n` → `v1.n+1`. Schema first, server second, wait 10 minutes, repeat. Skipping a minor version risks data incompatibility.**

## Sequential Upgrades Are Mandatory

> Temporal Server should be upgraded sequentially. If you're on version `v1.n.x`, your next upgrade should be to `v1.n+1.x`.

Backward compatibility is guaranteed only between consecutive minor releases. Jumping `v1.18 → v1.22` is unsupported and may corrupt persistence.

## Per-Hop Procedure

For each minor-version step:

1. **Update schema** with the appropriate DB tool. Schema must precede server.
2. **Roll the server** to the new version (rolling restart across history / matching / frontend / worker pods).
3. **Wait 10 minutes.** History service must reload all shards and complete metadata updates.
4. **Verify** — sync match rate, persistence latency, no error spike.
5. **Repeat** with the next minor version until target reached.

⚠️ Skipping the 10-minute wait can cause shard ownership churn and persistence errors during the next hop.

## Schema Tools

### PostgreSQL — Default Store

```bash
./temporal-sql-tool \
  --tls --tls-enable-host-verification \
  --tls-cert-file <path> --tls-key-file <path> --tls-ca-file <path> \
  --ep localhost -p 5432 -u temporal -pw temporal \
  --pl postgres --db temporal \
  update-schema -d ./schema/postgresql/v96/temporal/versioned
```

### PostgreSQL — Visibility (v12+)

```bash
./temporal-sql-tool \
  --tls --tls-enable-host-verification \
  --tls-cert-file <path> --tls-key-file <path> --tls-ca-file <path> \
  --ep localhost -p 5432 -u temporal -pw temporal \
  --pl postgres12 --db temporal_visibility \
  update-schema -d ./schema/postgresql/v12/visibility/versioned
```

### MySQL

Same `temporal-sql-tool` with `--pl mysql8` and the MySQL schema directory.

### Cassandra (default + visibility)

```bash
temporal-cassandra-tool \
  --tls --user <user> --password <pwd> \
  --endpoint <host> --keyspace temporal --timeout 120 \
  update --schema-dir ./schema/cassandra/temporal/versioned

temporal-cassandra-tool \
  --tls --user <user> --password <pwd> \
  --endpoint <host> --keyspace temporal_visibility --timeout 120 \
  update --schema-dir ./schema/cassandra/visibility/versioned
```

### Elasticsearch Visibility

```bash
temporal-elasticsearch-tool \
  --endpoint "$ES_SCHEME://$ES_HOST:$ES_PORT" \
  update-schema --index "$ES_VISIBILITY_INDEX"
```

⚠️ Schema tools are **forward-only**. They apply versioned migrations idempotently, but there is no managed rollback — restore from backup if you need to revert.

## Rolling Restart

Within a minor version, services should restart in rolling fashion to maintain availability ([[Temporal SH 05 - Production Checklist]] target 99.99%):

- Drain one pod / instance at a time.
- Wait for membership ring to stabilize.
- Move to the next.

Order is flexible within a hop, but a common pattern is: history → matching → frontend → worker.

## Helm Compatibility Reminder

⚠️ Temporal Helm chart **0.73.1+** is required for server **1.30+**. Update the chart before bumping server image. See [[Temporal SH 02 - Deployment]].

## Pre-Production Validation

- Run the same upgrade against a staging cluster that has been replayed with production-like load.
- Watch for instability (shard reassignment, persistence errors, sync match drops) for at least an hour after each hop.
- Verify [[Temporal RF 03 - Events]] — new event types added in the target version should serialize / deserialize correctly.
- Confirm worker SDK versions remain compatible (Temporal commits to N-2 SDK compatibility per server release; check release notes).

## Downtime Considerations

- Rolling restart on properly clustered persistence (Cassandra multi-DC, RDS multi-AZ) → effectively zero downtime.
- Schema migrations are online for SQL (DDL is non-blocking for the relevant tables) and online for ES.
- Single-node persistence in dev / Compose → expect a brief outage during the schema apply.

## Per-Hop Checklist

- [ ] Read release notes for the target version (look for breaking changes).
- [ ] Snapshot DB before the hop.
- [ ] Run schema tool against staging; verify success.
- [ ] Apply schema in production.
- [ ] Roll server to the target image.
- [ ] Wait 10 minutes; verify metrics.
- [ ] Repeat for the next minor version.

💡 **Takeaway:** Schema first, server second, wait, repeat — never skip a minor version. Pin the matching Helm chart, validate in staging, and treat each hop as an independent release with its own monitoring window.
