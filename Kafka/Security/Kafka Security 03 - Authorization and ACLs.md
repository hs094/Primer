# 03 — Authorization and ACLs

🔑 `StandardAuthorizer` ([[KRaft]]) gates every op via `(Principal, Operation, Resource, Host)` tuples.

Source: https://kafka.apache.org/42/security/authorization-and-acls/

## Enable Authorization

```properties
# KRaft mode (4.x)
authorizer.class.name=org.apache.kafka.metadata.authorizer.StandardAuthorizer

# legacy ZK mode (pre-4.x): kafka.security.authorizer.AclAuthorizer
super.users=User:admin;User:ops
allow.everyone.if.no.acl.found=false
```

## ACL Rule Shape

> "Principal **P** is [Allowed|Denied] Operation **O** From Host **H** on Resource **R**"

- Default-deny: a resource with **no ACLs** is accessible only by `super.users`.
- ⚠️ `allow.everyone.if.no.acl.found=true` flips this — broad fallback, dangerous in multi-tenant.
- Deny rules win over allow rules.

## Resources

| Resource | Examples of ops it gates |
|---|---|
| **Topic** | Read, Write, Create, Delete, Describe, AlterConfigs, DescribeConfigs |
| **Group** | Read (consume), Describe, Delete |
| **Cluster** | ClusterAction (replication), AlterConfigs, IdempotentWrite, Alter, Describe |
| **TransactionalId** | Write, Describe (for txn producers) |
| **DelegationToken** | Describe |
| **User** | CreateTokens, DescribeTokens |

## Operations

`Read`, `Write`, `Create`, `Delete`, `Alter`, `Describe`, `ClusterAction`, `DescribeConfigs`, `AlterConfigs`, `IdempotentWrite`, `CreateTokens`, `DescribeTokens`, `All`.

| Op | Where it applies |
|---|---|
| `Read` | Consumer fetch from topic; offset commit on group |
| `Write` | Producer send |
| `IdempotentWrite` | Idempotent producer (cluster-level) |
| `ClusterAction` | Inter-broker replication (broker→broker) |
| `Describe` | Metadata visibility |

## Principals

`User:bob` — type is `User` by default, name comes from the auth layer:

- [[mTLS]]: cert subject DN (mappable via `ssl.principal.mapping.rules`).
- [[SASL]]/PLAIN, SCRAM: SASL username.
- Kerberos: short principal.

## `kafka-acls.sh` Examples

```bash
# allow Bob to read topic Test-topic
kafka-acls.sh --bootstrap-server localhost:9092 --add \
  --allow-principal User:Bob --operation Read --topic Test-topic

# producer convenience: Write+Describe on topic, IdempotentWrite on cluster
kafka-acls.sh ... --add --allow-principal User:Alice --producer --topic orders

# consumer convenience: Read+Describe on topic + Read on group
kafka-acls.sh ... --add --allow-principal User:Alice --consumer --topic orders --group analytics

# prefixed ACL — covers a whole namespace
kafka-acls.sh ... --add --allow-principal User:teamA \
  --producer --resource-pattern-type prefixed --topic teamA.

# host restriction
kafka-acls.sh ... --add --allow-principal User:Bob \
  --allow-host 10.0.0.5 --operation Read --topic Test-topic

# list / remove
kafka-acls.sh ... --list --topic Test-topic
kafka-acls.sh ... --remove --allow-principal User:Bob --operation Read --topic Test-topic
```

## Tips

- 💡 `--producer` and `--consumer` flags expand to the right multi-resource ACL set — use them.
- 💡 Use **prefixed** resource patterns for tenants — single rule per tenant prefix.
- ⚠️ Forgetting Group `Read` is the #1 reason "consumer can fetch but offsets won't commit".
- ⚠️ `IdempotentWrite` on `Cluster` is required for idempotent/transactional producers — easy to miss.

## Tags
[[Kafka]] [[ACL]] [[Security]] [[KRaft]]
