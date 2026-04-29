# NATS — INDEX

🔑 [[NATS]] is a high-performance subject-based [[Messaging]] system: cheap pub/sub at the core, durable streams + KV + object store via [[JetStream]] on top.

Source: https://docs.nats.io/

## Reading Order
1. [[NATS 01 - Overview]] — what NATS is, perf profile, when to pick it.
2. [[NATS 02 - Subjects]] — the only routing primitive: dot-hierarchies + `*` / `>` wildcards.
3. [[NATS 03 - Pub Sub]] — fire-and-forget core, contrasted with [[Kafka]].
4. [[NATS 04 - Request Reply]] — RPC over inbox subjects.
5. [[NATS 05 - Queue Groups]] — server-side load balancing across N subscribers.
6. [[NATS 06 - JetStream]] — streams + consumers, retention policies, at-least-once / exactly-once.
7. [[NATS 07 - Key Value Store]] — JetStream-backed KV with watch + history (vs [[Redis]]).
8. [[NATS 08 - Object Store]] — JetStream-backed blob store (vs S3).
9. [[NATS 09 - Clusters and Leaf Nodes]] — cluster, super-cluster, [[Edge]] leaf nodes.
10. [[NATS 10 - Security]] — TLS, NKey, JWT, accounts, per-subject permissions.

## Mental Model
| Layer | Gives you | Cost |
|---|---|---|
| Core | pub/sub, request/reply, queue groups | ephemeral, at-most-once |
| JetStream | streams, KV, object store | disk + replication |
| Cluster / Leaf | HA, geo, edge reach | ops complexity |
| Accounts + JWT | multi-tenant isolation | key management |

## Cross-Refs
- vs [[Kafka]]: bus vs log — see notes 03, 05, 06.
- vs [[Redis]]: see note 07 (KV).
- [[Edge]] / IoT: notes 01, 08, 09.

## Tags
[[NATS]] [[JetStream]] [[Pub Sub]] [[Messaging]] [[Edge]]
