# 05 — Quotas

🔑 Per-(user, client-id) bandwidth and request-rate limits enforced at the broker via channel muting — protects the cluster from a noisy neighbor.

Source: https://kafka.apache.org/42/design/design/

## Two Kinds
| Quota | Unit | Since |
|---|---|---|
| Bandwidth | bytes/sec (in / out) | 0.9 |
| Request rate | % broker CPU / network thread time | 0.11 |
- Bandwidth caps producer/consumer throughput; request quota caps how much CPU a client can monopolize.

## Identity
- Quota tuple is `(user, client-id)`.
- User comes from authenticated principal; client-id is a free-form producer/consumer config string.
- ⚠️ With no auth, all users collapse to the unauthenticated principal — quota effectively becomes per-client-id only.

## 8-Level Precedence
Most specific match wins:
1. `(user, client-id)` — exact pair
2. `(user, default-client-id)` — user with any client
3. `user` — flat user limit
4. `(default-user, client-id)` — any user, specific client
5. `(default-user, default-client-id)`
6. `default-user`
7. `client-id`
8. `default-client-id`
- 💡 Defaults at any level apply when no more specific override exists.

## Throttling Mechanism
- Broker measures usage in a sliding window (default: 30 windows × 1 s = 30 s).
- When a request would exceed the rate, broker:
  1. Computes the delay needed to bring rate under cap.
  2. Sends the response **immediately** (with throttle-time hint in the response header).
  3. Mutes the client's channel for the delay so no further requests are processed.
- Client sees latency, not errors. New 0.9 clients self-throttle on the hint; older clients just see queueing.
- ⚠️ Closing the connection wouldn't help (TCP buffers refill from page cache faster than they drain).

## Why Multi-Window
- Smooths bursts — a single 1 s window allows large spikes; a 30 s rolling window enforces sustained rate.
- Trade-off: longer windows tolerate longer bursts but recover slowly when client misbehaves.

## Tags
[[Kafka]] [[Quotas]] [[Throttling]] [[Multi-Tenancy]] [[Brokers]]
