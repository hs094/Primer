# 06 — Multi-Tenancy

🔑 Isolate tenants via **prefixed names + ACLs + quotas**. No tenant should starve another.

Source: https://kafka.apache.org/42/operations/multi-tenancy/

## Topic Namespacing

Hierarchical convention, e.g.

```
<org>.<team>.<dataset>.<event>
acme.payments.orders.created
```

- Enforce via prefixed [[ACL]]s on `Create`.
- Disable wild-west creation: `auto.create.topics.enable=false`.
- For strict shape: implement a custom `CreateTopicPolicy`.

## Per-Tenant ACLs

Use **prefixed ACLs** so one rule covers a whole namespace:

```bash
kafka-acls.sh --add --allow-principal User:Alice --producer \
  --resource-pattern-type prefixed --topic acme.infosec.
```

- Principals scoped per tenant (`User:team-a`, `User:team-b`).
- Combine with [[SASL]]/[[mTLS]] to authenticate the principal.

## Quotas

Cap each tenant's broker resources:

| Quota Type | Config | Purpose |
|---|---|---|
| Bandwidth | `producer_byte_rate`, `consumer_byte_rate` | Network fairness |
| Request rate | `request_percentage` | CPU on broker I/O threads |
| Connection | `connection_creation_rate` | Slow connection floods |
| Controller mutation | `controller_mutation_rate` | Cap topic/partition churn |

Set per `users`, `clients`, or both. See [[Quotas]].

## Transactional Reads

`isolation.level` on consumer:

| Value | Behavior |
|---|---|
| `read_uncommitted` (default) | Sees all records including aborted txn data. |
| `read_committed` | Only commits visible — required when tenants share a topic with transactional producers. |

## Capacity Planning

- ⚠️ **Partition count** is a cluster-wide budget — each consumes file handles, metadata replication cost, and controller CPU.
- Plan a per-tenant partition quota up front; total against cluster limits (rough rule: ≤ 200K partitions per controller quorum).
- Per-topic `retention.ms` / `retention.bytes` to bound disk per tenant.

## Other Safeguards

- 💡 Schema registry per tenant or namespace for contract enforcement.
- Monitor quota throttle metrics — tenants hitting caps cause client-side latency, not broker errors.
- Consider [[Tiered Storage]] to give tenants long retention without disk burden.

## Tags
[[Kafka]] [[ACL]] [[Quotas]] [[Operations]]
