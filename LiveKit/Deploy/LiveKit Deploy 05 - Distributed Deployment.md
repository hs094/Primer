# 05 — Distributed Deployment

🔑 Configure Redis and LiveKit auto-clusters: any node can receive signaling, route to the room-hosting node, drain on SIGTERM, and pick geo-nearest nodes for new rooms — but a single room still must fit on one node.

Source: https://docs.livekit.io/home/self-hosting/distributed/

## Architecture

- **Redis** is the cluster bus + shared store. Presence of `redis:` config flips LiveKit into distributed mode automatically.
- **Signaling bridge**: client WebSocket lands on any node; that node proxies signaling to the node hosting the room.
- **Room placement**: on `CreateRoom`, the receiving node picks an available node (low `sysload`, region-near) to host.
- **Single-node room cap**: rooms don't span nodes. Scale horizontally by spreading rooms, not by sharding one room.

## Config

```yaml
# livekit.yaml on every node
port: 7880
bind_addresses: [""]
rtc:
  tcp_port: 7881
  port_range_start: 50000
  port_range_end: 60000
  use_external_ip: true

redis:
  address: redis.internal:6379
  username: ""
  password: ""
  db: 0
  # for sentinel / cluster:
  # sentinel_addresses: ["s1:26379", "s2:26379"]
  # sentinel_master_name: "mymaster"

region: us-east-1   # geo-aware routing

keys:
  APIxxx: secretxxx

turn:
  enabled: true
  domain: turn.example.com
  tls_port: 5349
  udp_port: 3478
  external_tls: true
```

## Required Ports per Node

| Port | Proto | Purpose |
|------|-------|---------|
| 443 / 7880 | TCP | Signaling (WSS) |
| 7881 | TCP | ICE/TCP |
| 50000-60000 | UDP | RTP media |
| 5349 | TCP | TURN/TLS |
| 3478 | UDP | TURN/UDP |
| 6379 | TCP | Redis (internal only) |

## TURN

Embedded TURN server reduces NAT/firewall failures:

- **TURN/TLS on 5349** (or 443 without an LB) — looks like HTTPS, traverses corporate firewalls.
- **TURN/UDP on 443** — preferred for latency.

Requires a real domain + cert from a trusted CA. Self-signed will not work.

## Graceful Downscaling

Node receives `SIGTERM` / `SIGINT` / `SIGQUIT`:
1. Marks itself draining in Redis.
2. Stops accepting new room creations.
3. Existing rooms continue until participants leave.
4. Process exits when room count hits zero.

Set Kubernetes `terminationGracePeriodSeconds` long enough for typical session length, or accept hard cutoffs.

## Multi-Region

Set `region:` per node. Routing prefers:
1. Nodes with `sysload < sysload_limit`.
2. Nodes in the region nearest the client's signaling endpoint.

Front signaling with a geo-DNS or anycast LB so users land on a regional node first.

## Scalability Notes

- Bound by **CPU + bandwidth**. Use compute-optimized instances; 10 Gbps NIC for production.
- Use **host networking** in Docker — bridged networking breaks WebRTC ICE.
- Redis must be HA (Sentinel or managed) — it's the cluster's single point of state.
- Enable Prometheus on every node, scrape with a single Prom server.

## Tags
[[LiveKit]] [[Deploy]] [[Self-Hosting]] [[Distributed]]
