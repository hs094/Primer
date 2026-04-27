# UG 10 — Security

🔑 Celery trusts whatever the broker hands it — the broker, not Celery, is your security boundary, and `pickle` makes that boundary a remote-code-execution channel.

## Threat model

Three surfaces:

- **Broker** — workers consume and trust messages by default. "The broker is guarded from unwanted access, especially if accessible to the public." Firewall it, scope credentials, and use TLS via `broker_use_ssl`.
- **Client** — anything that can publish to the broker can run arbitrary tasks. If the client host is compromised and the broker is open, all bets are off.
- **Worker** — runs with the privileges of whoever started it. Restrict with a dedicated unix user, subprocess isolation (`fork()` + `execve()`), and filesystem confinement (chroot, jail, container, VM).

⚠️ Never run workers as root unless you have a very specific reason. Task code that calls `subprocess` or writes files inherits that privilege.

## Serializers — the pickle problem

`pickle` can deserialize arbitrary Python objects, which means a malicious payload can execute code at unpickle time. Quoting the docs: pickle is **inherently insecure**.

JSON has been the default since Celery 4.0. Lock the worker down to known-safe serializers via an allowlist:

```python
app.conf.update(
    accept_content=['json'],          # by short name
    # or by content-type:
    # accept_content=['application/json'],
    task_serializer='json',
    result_serializer='json',
)
```

`accept_content` is enforced on the worker — even if a client sends pickle, an unlisted content-type is rejected before deserialization.

## Message signing with X.509

For multi-tenant brokers or any case where you cannot fully trust the transport, sign messages with public-key crypto. Each worker has its own keypair; certificates are exchanged out-of-band into a shared store.

```python
from celery import Celery

app = Celery()
app.conf.update(
    security_key='/etc/ssl/private/worker.key',
    security_certificate='/etc/ssl/certs/worker.pem',
    security_cert_store='/etc/ssl/certs/*.pem',
    security_digest='sha256',
    task_serializer='auth',
    event_serializer='auth',
    accept_content=['auth'],
)
app.setup_security()
```

Config keys:

- `security_key` — PEM-encoded private key for the local node.
- `security_certificate` — public cert for the local node (signs outbound messages).
- `security_cert_store` — glob matching trusted peer certificates (verifies inbound).
- `security_digest` — signing algorithm; use `sha256`.

`app.setup_security()` registers the `auth` serializer **and disables all insecure serializers** in one shot. Call it before the worker starts consuming.

The `auth` serializer wraps an inner serializer (default JSON) — the payload is still JSON, just signed.

## Broker access control

### RabbitMQ

Per-vhost users with explicit permissions:

```bash
rabbitmqctl add_vhost /myapp
rabbitmqctl add_user celery 's3cret'
rabbitmqctl set_permissions -p /myapp celery '.*' '.*' '.*'
```

Tighten the `configure / write / read` regexes to the actual exchanges and queues the role needs. Production workers rarely need `configure` once topology is provisioned.

### Redis

Use ACLs (Redis 6+) to scope a Celery user to the keys it actually touches:

```
ACL SETUSER celery on >s3cret ~celery* ~unacked* ~_kombu* &* +@all -@dangerous
```

Bind Redis to localhost or a private network interface (`bind 127.0.0.1`) and require `requirepass` even on internal networks. Disable destructive commands (`rename-command FLUSHDB ""`) on shared instances.

### TLS to the broker

```python
import ssl
app.conf.broker_use_ssl = {
    'keyfile': '/etc/ssl/celery.key',
    'certfile': '/etc/ssl/celery.crt',
    'ca_certs': '/etc/ssl/ca.crt',
    'cert_reqs': ssl.CERT_REQUIRED,
}
```

Use `redis_backend_use_ssl` for the result backend — they are separate.

## Operational hygiene

- Centralize logs via syslog so a compromised host cannot silently rewrite its own audit trail.
- File integrity monitoring on worker hosts — OSSEC, Samhain, AIDE, Tripwire.
- Rotate broker credentials and signing keys; `security_cert_store` accepts a glob, so adding a new peer cert is a deploy without restart of every node.
- Keep `accept_content` minimal — adding `pickle` "just in case" defeats the rest of this page.

See [[Celery UG 04 - Calling Tasks]] for serializer selection per call and [[Celery UG 11 - Optimizing]] for `broker_pool_limit` interactions with TLS.

💡 Default to JSON, allowlist content types, and only reach for `auth`/X.509 when you genuinely cannot trust the broker — but never trust pickle on a network you do not control.
