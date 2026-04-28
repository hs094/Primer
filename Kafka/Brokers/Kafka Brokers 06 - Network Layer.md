# 06 — Network Layer

🔑 A standard NIO server with one acceptor + N processor threads — kept deliberately simple so clients in any language can speak the protocol.

Source: https://kafka.apache.org/42/implementation/network-layer/

## Threading Model
- **1 acceptor thread** — accepts TCP connections, hands them off.
- **N processor threads** — each owns a fixed set of connections and drives reads/writes via Java NIO `Selector`.
- Behind the processors sits a request handler pool that does the actual work (log appends, fetches, metadata).
- Battle-tested pattern; design choice favors clarity over novelty.

## Sendfile Plumbing
- The `TransferableRecords` interface exposes a `writeTo(channel)` method.
- File-backed message sets implement this with `FileChannel.transferTo` → kernel-level [[Zero-Copy]] via `sendfile`.
- Lets the broker stream a fetch response straight from page cache to the client socket without bouncing through a user-space buffer.

## Protocol Posture
- Binary protocol over plain TCP (length-prefixed request/response).
- Kept intentionally simple to ease porting to non-JVM clients (Go, Rust, Python, C++ all have native implementations).
- Versioned per API key — brokers handle multiple versions concurrently for rolling upgrades.

## Tags
[[Kafka]] [[Network Layer]] [[NIO]] [[sendfile]] [[Brokers]]
