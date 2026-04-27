# GS 02 — Backends and Brokers

🔑 The **broker** moves messages between client and workers; the **backend** stores task results. They're independent — pick each based on what you actually need.

## Broker comparison

| Name        | Status       | Monitoring | Remote Control |
|-------------|--------------|------------|----------------|
| RabbitMQ    | Stable       | Yes        | Yes            |
| Redis       | Stable       | Yes        | Yes            |
| Amazon SQS  | Stable       | No         | No             |
| Zookeeper   | Experimental | No         | No             |
| Kafka       | Experimental | No         | No             |
| GC Pub/Sub  | Experimental | Yes        | Yes            |

⚠️ SQS and Kafka **lack remote control** — `celery inspect`, `celery control`, and Flower's live actions won't work. Pick RabbitMQ or Redis if you need them.

## RabbitMQ

The default and most fully-featured broker. Stable, supports remote control, supports monitoring events. Best when you have **larger messages** or need richer routing semantics. Scales to high throughput but you'll need to think about cluster sizing eventually.

```bash
$ sudo apt-get install rabbitmq-server
$ docker run -d -p 5672:5672 rabbitmq
```

Broker URL: `pyamqp://guest@localhost//`

## Redis

Fast K/V store. Doubles as a great **result backend** because it's fast at small reads. As a broker it handles small messages well but can congest with very large payloads.

```bash
$ docker run -d -p 6379:6379 redis
```

Broker URL: `redis://localhost`

The "RabbitMQ as broker + Redis as backend" combo is very commonly used.

## Amazon SQS

Stable but no monitoring, no remote control. Useful if you're already deep in AWS and want a managed queue with no ops.

## Experimental brokers

Zookeeper, Kafka, Google Cloud Pub/Sub — usable, but expect rough edges and missing features.

## Result backends

Results are off by default. When you turn them on, options include:

- **Redis** — "super fast K/V store, making it very efficient for fetching the results of a task call." Default pick for most stacks.
- **RabbitMQ (`rpc://`)** — uses temporary per-client queues. Fine for short-lived results; not for durable history.
- **SQLAlchemy** — MySQL, PostgreSQL, SQLite, etc. Use when you want results queryable from the same DB as your app data.
- **Cassandra** — when you want long-term, distributed persistence.
- **Custom** — you can plug in your own.

For long-term persistence prefer **PostgreSQL / MySQL / Cassandra** over Redis or RabbitMQ-RPC.

## Picking, in practice

- Local dev / small prod: **RabbitMQ broker + Redis backend**.
- Need durable result history: keep RabbitMQ as broker, switch backend to **PostgreSQL** (via SQLAlchemy).
- Already on AWS, no need for live control: **SQS** + whatever backend you like.
- Heavy small-message throughput: **Redis broker**.
- Heavy large-message throughput / complex routing: **RabbitMQ broker**.

💡 Broker and backend are **separate** decisions — don't conflate them. Wiring is just the two URLs in `Celery(broker=..., backend=...)`. Next: [[Celery GS 03 - First Steps with Celery]].
