# 08 — Consumer Rebalance Protocol (KIP-848)

🔑 Broker computes assignments incrementally — no more stop-the-world group syncs.

Source: https://kafka.apache.org/42/operations/consumer-rebalance-protocol/

## What Changed

| | Classic | New (KIP-848) |
|---|---|---|
| Assignment computed by | **Group leader (a client)** | **Broker (group coordinator)** |
| Sync model | All members pause, sync barrier | **Heartbeat-based, incremental** |
| Stop-the-world | Yes | No |
| Client logic | Heavy (assignors live in client) | Thin (just heartbeat + ack) |

The new protocol is "fully incremental" — only members whose assignment actually changes do work.

## Server-Side Knobs

These now live on the broker, not the client:

| Config | Was (classic, client) | Now (new, broker) |
|---|---|---|
| Heartbeat | `heartbeat.interval.ms` | `group.consumer.heartbeat.interval.ms` |
| Session timeout | `session.timeout.ms` | `group.consumer.session.timeout.ms` |
| Assignor | `partition.assignment.strategy` | `group.consumer.assignors` |

💡 Operators can change rebalance behavior cluster-wide without redeploying clients.

## Opt-In

```
group.protocol=consumer        # new protocol
group.protocol=classic         # old protocol (default for legacy clients)
```

- Available from Kafka **4.0** on brokers; opt-in per consumer.
- Empty groups auto-convert when first member joins with a new protocol.
- Mixed-protocol groups are supported during rolling upgrade.

## Limitations

- ⚠️ **Client-side custom assignors not supported** — must use server-provided ones.
- ⚠️ Rack-aware assignment incomplete (KAFKA-17747).

## Migration Path

1. Brokers on 4.x already support both protocols.
2. Roll consumers with `group.protocol=consumer`.
3. Group converts when fully drained or via online interop.

## Tags
[[Kafka]] [[Consumers]] [[Operations]]
