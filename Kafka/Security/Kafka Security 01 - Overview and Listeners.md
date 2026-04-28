# 01 — Security Overview and Listeners

🔑 Three layers (encryption / authentication / authorization), wired up via **listeners + protocol map**.

Sources:
- https://kafka.apache.org/42/security/security-overview/
- https://kafka.apache.org/42/security/listener-configuration/

## Three Layers

| Layer | Tech |
|---|---|
| **Encryption** (in transit) | [[SSL]]/TLS |
| **Authentication** | [[SSL]] client certs ([[mTLS]]) or [[SASL]] |
| **Authorization** | [[ACL]] via `StandardAuthorizer` (KRaft) |

💡 Security is **opt-in and granular** — you can mix authenticated, unauthenticated, encrypted, and plaintext clients on the same cluster, on different listeners.

## Security Protocols

| Protocol | Encryption | Auth |
|---|---|---|
| `PLAINTEXT` | none | none |
| `SSL` | TLS | optional [[mTLS]] |
| `SASL_PLAINTEXT` | none | [[SASL]] |
| `SASL_SSL` | TLS | [[SASL]] (creds protected by TLS) |

⚠️ `SASL_PLAINTEXT` sends credentials over the wire in cleartext for `PLAIN` mechanism — only acceptable inside trusted nets.

## Listener Configuration

Listeners are named and mapped to protocols:

```properties
# named listeners + bind addresses
listeners=CLIENT://0.0.0.0:9092,BROKER://0.0.0.0:9093,CONTROLLER://0.0.0.0:9094

# what clients see (DNS / public)
advertised.listeners=CLIENT://broker1.example.com:9092,BROKER://broker1.internal:9093

# name -> protocol
listener.security.protocol.map=CLIENT:SASL_SSL,BROKER:SSL,CONTROLLER:SSL

# which listener carries inter-broker traffic
inter.broker.listener.name=BROKER

# KRaft only: which listener(s) controllers serve on
controller.listener.names=CONTROLLER
```

## Key Properties

| Property | Purpose |
|---|---|
| `listeners` | Bind sockets — what the broker listens on |
| `advertised.listeners` | What brokers tell clients in metadata responses |
| `listener.security.protocol.map` | Maps listener **name** → wire protocol |
| `inter.broker.listener.name` | Which listener handles broker↔broker |
| `security.inter.broker.protocol` | Legacy alternative (mutually exclusive with above) |
| `controller.listener.names` | KRaft controller endpoint |

## Gotchas

- ⚠️ `advertised.listeners` must be the address **clients can actually reach** — wrong value = "broker is up but clients can't connect".
- ⚠️ Controller listener cannot share a name with `inter.broker.listener.name` in [[KRaft]].
- 💡 Prefer **explicit listener names** (`CLIENT`, `BROKER`) over reusing protocol names — clearer intent, safer to reconfigure.
- One listener per port; different protocols → different ports.

## Tags
[[Kafka]] [[Security]] [[SSL]] [[SASL]] [[mTLS]]
