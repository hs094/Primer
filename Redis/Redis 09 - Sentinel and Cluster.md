# 09 — Sentinel and Cluster

🔑 Two different problems, two different tools. Sentinel = HA + auto-failover for a single dataset. Cluster = horizontal sharding across nodes.

Source: https://redis.io/docs/latest/operate/oss_and_stack/management/sentinel/, https://redis.io/docs/latest/operate/oss_and_stack/management/scaling/

## Sentinel — High Availability
- Separate process (port 26379) that monitors masters/replicas and runs leader election.
- Three responsibilities: **monitor**, **notify**, **automatic failover**, **config provider** (clients ask Sentinel for current master).
- Run **≥3** Sentinels across independent failure domains.

```conf
sentinel monitor mymaster 10.0.0.1 6379 2     # quorum=2
sentinel down-after-milliseconds mymaster 5000
sentinel failover-timeout mymaster 60000
sentinel parallel-syncs mymaster 1
```

| Term | Meaning |
|---|---|
| **SDOWN** | Subjectively down — one Sentinel can't reach master |
| **ODOWN** | Objectively down — `quorum` Sentinels agree |
| **Quorum** | Min Sentinels to mark ODOWN |
| **Majority** | Required to *authorize* failover (separate from quorum) |

- ⚠️ Async replication = failover can drop writes from the old master. Mitigate with `min-replicas-to-write` + `min-replicas-max-lag`.

## Redis Cluster — Sharding
- Built-in sharding across N masters, each with its own replicas.
- Keyspace is split into **16384 hash slots**; each node owns a contiguous range.
- Slot for a key = `CRC16(key) mod 16384`.

```bash
redis-cli --cluster create \
  10.0.0.1:7000 10.0.0.2:7000 10.0.0.3:7000 \
  10.0.0.1:7001 10.0.0.2:7001 10.0.0.3:7001 \
  --cluster-replicas 1
```

### Client Redirects
| Reply | Meaning |
|---|---|
| `MOVED <slot> <ip:port>` | Permanent: slot now owned elsewhere; client updates its slot map |
| `ASK <slot> <ip:port>` | Transient (during slot migration): retry once with `ASKING` prefix |

A cluster-aware client (redis-py `RedisCluster`, ioredis Cluster, go-redis `ClusterClient`) handles both transparently.

### Multi-Key Ops in Cluster
- All keys touched by one command must live on the **same slot**.
- Force colocation with **hash tags**: `user:{42}:profile` and `user:{42}:cart` both hash on `42`.
- ⚠️ `MGET k1 k2` across slots → `CROSSSLOT` error.

## When To Use What
| Need | Pick |
|---|---|
| Single dataset fits in RAM, want auto-failover | Sentinel |
| Dataset > one box, or hot-key load > one core | Cluster |
| Both | Cluster (each shard is auto-failover by design) |

## Tags
[[Redis]] [[Sentinel]] [[Redis Cluster]] [[Replication]] [[Hash Slots]]
