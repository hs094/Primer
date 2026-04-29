# 02 — ClickHouse Overview

🔑 [[Clickhouse]] is a **distributed columnar OLAP DBMS** built for billions-of-rows-per-second scans. Yandex (now ClickHouse Inc) open-sourced it in 2016.

Source: https://clickhouse.com/docs/intro

## Headline
- ~1 billion rows/sec aggregation on a single node (in their docs example: 100M rows in 92 ms).
- Native SQL, async multi-master replication, role-based ACL.
- Runs as a server (binary or Docker), or ClickHouse Cloud.

## The MergeTree Family
[[MergeTree]] is the workhorse engine; everything else is a variant.
- Data lives in **parts** on disk (sorted, compressed, columnar).
- Background **merges** combine parts → fewer, larger files → better compression + query speed.
- Inserts → new part. Reads → fan-out across parts.

## `ORDER BY` = Primary Index
🔑 The `ORDER BY` clause defines both physical sort order **and** the (sparse) primary index.
```sql
CREATE TABLE events (
    user_id UInt64,
    event_time DateTime,
    event_type LowCardinality(String),
    payload String
) ENGINE = MergeTree
ORDER BY (user_id, event_time)
PARTITION BY toYYYYMM(event_time)
TTL event_time + INTERVAL 90 DAY;
```
- Index is **sparse**: one entry per granule (8192 rows by default), fits in RAM.
- Filter prefix of `ORDER BY` = blazing. Filter on suffix-only column = full scan.

## TTL — automatic data lifecycle
`TTL event_time + INTERVAL 90 DAY` → background merge drops expired rows. Can also recompress, move to cold tier (S3), or aggregate.

## Partitions
Logical grouping (e.g. by month). Used for **partition pruning** + bulk drop. Don't over-partition: too many small parts kills merges.

⚠️ Single-row `UPDATE`/`DELETE` are async, async, async (`ALTER TABLE ... UPDATE` runs as a background mutation). Not OLTP.

## Tags
[[Clickhouse]] [[MergeTree]] [[Columnar]] [[OLAP]] [[Parts]]
