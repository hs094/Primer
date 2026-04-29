# 09 — Streaming Replication and HA

🔑 Physical replication ships WAL bytes from primary to standby — cluster-wide, read-only standbys, foundation for HA.

Source: https://www.postgresql.org/docs/current/warm-standby.html

## How It Works
- Primary writes WAL → standby's `walreceiver` streams it via replication protocol → `startup` process applies it.
- **Hot standby**: read-only queries while replay continues.
- All databases / extensions / users replicate atomically (it's byte-level).

## Setup Sketch
```bash
# Standby init
pg_basebackup -h primary -D /var/lib/postgresql/data \
  -R -P -X stream -c fast -S standby1
```
- `-R` writes `standby.signal` + `primary_conninfo`.
- `-S` uses a named replication slot to retain WAL.

## Sync vs Async
| Mode | Setting | Trade-off |
|---|---|---|
| Async | (default) | Lowest latency, can lose committed tx on failover |
| Sync | `synchronous_commit = on` + `synchronous_standby_names` | No data loss; commits stall if standby down |
| `remote_apply` | Strongest | Reads on standby see committed writes immediately |

⚠️ With one sync standby and it goes offline, every commit hangs unless `synchronous_standby_names` lists alternates (`ANY 1 (s1, s2, s3)`).

## Replication Slots
- Persist the standby's position so primary keeps WAL.
- ⚠️ Disconnected slot still pins WAL → disk fills. Monitor `pg_replication_slots`.

## Failover Tools
| Tool | Notes |
|---|---|
| **Patroni** | Etcd/Consul/ZK-backed, leader election, REST API. Industry standard. |
| **pg_auto_failover** | Citus/MS-backed, monitor + keeper architecture. |
| **repmgr** | Older, manual-leaning. |
| Cloud (RDS, Aurora, CloudSQL) | Failover handled by provider. |

## Promotion
```sql
-- On the chosen standby
SELECT pg_promote();
```
- After promotion, original primary must be re-cloned or rewound (`pg_rewind`) before rejoining.

## Read Scaling
- 💡 Standbys serve read traffic — but watch replication lag with `pg_stat_replication`.
- Sticky reads after writes need `read your own writes` handling (route to primary or use `synchronous_commit = remote_apply`).

## Tags
[[PostgreSQL]] [[Replication]] [[HA]] [[WAL]]
