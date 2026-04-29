# Redis — INDEX

🔑 Master index for the Redis primer. Each note is one focused topic, ≤50 lines, redis-py async first.

Source: https://redis.io/docs/latest/

## Notes
| # | Note | Section | Core idea |
|---|---|---|---|
| 01 | [[Redis 01 - Overview]] | Foundations | In-memory store, single-threaded, RESP |
| 02 | [[Redis 02 - Strings and Counters]] | Data types | `SET`/`INCR`/TTL/`SETNX` |
| 03 | [[Redis 03 - Lists Hashes Sets]] | Data types | Queues, records, membership |
| 04 | [[Redis 04 - Sorted Sets]] | Data types | Leaderboards, time windows, priority queues |
| 05 | [[Redis 05 - Streams]] | Messaging | Consumer groups, PEL, ack — vs Kafka |
| 06 | [[Redis 06 - Pub Sub]] | Messaging | Fire-and-forget, keyspace notifications |
| 07 | [[Redis 07 - Persistence]] | Operations | RDB vs AOF, fsync policies |
| 08 | [[Redis 08 - Replication]] | Operations | Async leader-follower, partial resync |
| 09 | [[Redis 09 - Sentinel and Cluster]] | Operations | HA + horizontal sharding |
| 10 | [[Redis 10 - Transactions and Scripts]] | Patterns | `MULTI`/`EXEC`, `WATCH`, Lua `EVAL` |
| 11 | [[Redis 11 - Pipelines and Performance]] | Patterns | RTT savings, benchmarking |
| 12 | [[Redis 12 - Vector Search]] | AI / Stack | FLAT/HNSW indexes, RedisVL |
| 13 | [[Redis 13 - Semantic Cache]] | AI / Stack | LLM cache by embedding similarity |
| 14 | [[Redis 14 - Client Libraries]] | Clients | redis-py, ioredis, go-redis, pool tuning |

## Reading Paths
- **Quick ramp**: 01 → 02 → 03 → 14.
- **Building a queue / event bus**: 03 → 05 → 06 → 10.
- **Going to production**: 07 → 08 → 09 → 11.
- **AI / RAG stack**: 12 → 13 → 14.

## Tags
[[Redis]] [[RESP]] [[Streams]] [[Sentinel]] [[Redis Cluster]] [[RedisVL]] [[Vector Search]] [[Semantic Cache]]
