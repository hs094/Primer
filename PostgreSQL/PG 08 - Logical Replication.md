# 08 — Logical Replication

🔑 Logical replication ships row-level changes via a publisher/subscriber model — selective, version-portable, the foundation of CDC.

Source: https://www.postgresql.org/docs/current/logical-replication.html

## Model
- **Publication** on the source — declares which tables/operations to publish.
- **Subscription** on the target — connects, creates a replication slot, replays changes.
- Decoded from WAL by an **output plugin** (`pgoutput` builtin; `wal2json` / `decoderbufs` for CDC tools).

```sql
-- Publisher
CREATE PUBLICATION app_pub FOR TABLE users, orders;

-- Subscriber
CREATE SUBSCRIPTION app_sub
  CONNECTION 'host=primary dbname=app user=repl password=...'
  PUBLICATION app_pub;
```

## REPLICA IDENTITY
Tells decoder how to identify the old row for `UPDATE`/`DELETE`.
| Setting | Old row data emitted |
|---|---|
| `DEFAULT` | Primary key columns |
| `USING INDEX idx` | A specific unique, non-deferrable, non-null index |
| `FULL` | Every column (expensive) |
| `NOTHING` | None — `UPDATE`/`DELETE` won't replicate |

```sql
ALTER TABLE big_table REPLICA IDENTITY FULL;
```

## Replication Slots
- Each subscription holds a slot on the publisher.
- Slot retains WAL until the subscriber confirms.
- ⚠️ Abandoned slot = WAL fills the disk. Drop with `SELECT pg_drop_replication_slot('name');`.

## vs Streaming (Physical) Replication
| Logical | Streaming |
|---|---|
| Row-level | Byte-level WAL |
| Subset of tables | Whole cluster |
| Cross-version | Same major version |
| Subscriber writable | Standby read-only |

## CDC with Debezium
- Debezium uses `pgoutput` (or `wal2json`) on a logical slot.
- Streams INSERT/UPDATE/DELETE to Kafka — feeds [[Kafka]] pipelines, search reindex, lakehouse.
- 💡 Set `REPLICA IDENTITY FULL` on tables where you need before-images.

## Conflicts
- Subscriber can hit unique violations / missing rows.
- ⚠️ Subscription pauses; you must resolve manually (`ALTER SUBSCRIPTION ... SKIP`).

## Tags
[[PostgreSQL]] [[Logical Replication]] [[CDC]] [[Kafka]]
