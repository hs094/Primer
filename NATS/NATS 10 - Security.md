# 10 — Security

🔑 Security is layered: **TLS** for transport, **NKey/JWT** for auth, **accounts** for tenant isolation, **per-subject permissions** for authz.

Source: https://docs.nats.io/running-a-nats-service/configuration/securing_nats/auth_intro

## Transport: TLS
- Mutual TLS supported — client certs can carry identity.
- Same cert/key story for client→server, route→route, gateway→gateway, leaf→hub.

## Authentication
| Mode | How | When to use |
|---|---|---|
| Token | shared secret in connect | quick dev, internal |
| User/password | bcrypted in config | small static deployments |
| TLS cert | client X.509 CN/SAN as identity | enterprise PKI |
| **NKey** | Ed25519 challenge-response, public key in config | secure, no shared secrets |
| **JWT** | signed user JWT + NKey signature, resolved via account JWT | decentralized, scales to ops |

💡 NKey: server stores only the **public** key; client signs a server-issued nonce with its private seed. No password ever crosses the wire.

## Accounts (Multi-Tenant Isolation)
- Account = isolated subject namespace. `foo` in account A is invisible to account B.
- **Exports / imports** open precise cross-account holes:
  - *Stream export* — share a pub/sub feed.
  - *Service export* — expose a request/reply endpoint.
- Three-tier hierarchy with JWT auth: **Operator → Account → User**.

## Authorization (Per-Subject)
```hocon
users = [
  { user: "billing", password: "...",
    permissions = {
      publish   = ["billing.events.>", "_INBOX.>"]
      subscribe = ["orders.created.>", "_INBOX.>"]
    }
  }
]
```
⚠️ Always allow `_INBOX.>` for any user that uses request/reply, otherwise replies get rejected.

## Tags
[[NATS]] [[JWT]] [[Messaging]]
