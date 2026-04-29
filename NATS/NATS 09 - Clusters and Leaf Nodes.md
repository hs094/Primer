# 09 — Clusters & Leaf Nodes

🔑 Three topology tiers: **cluster** (full mesh in one region), **super-cluster** (gateways across regions), **leaf nodes** (edge spokes).

Source: https://docs.nats.io/running-a-nats-service/configuration/clustering

## Cluster (Full Mesh)
- Every server connects to every other; gossip protocol auto-discovers peers from seed routes.
- One-hop forwarding rule: a route-received message is **never** re-forwarded across routes (prevents loops).
- Clients learn full server list and reconnect transparently on failure.
- Ports: client `4222`, route `6222`, monitoring `8222`.

## Super-Cluster (Gateways)
- Multiple clusters connected via **gateway** links (one per cluster pair).
- Optimised for WAN: per-account interest is propagated lazily, not full subscription state.
- Geo-affinity: queue-group msgs prefer local cluster members before crossing a gateway.

## Leaf Nodes
- A single `nats-server` running in **leaf mode** out at the edge or inside a third-party VPC.
- Outbound TLS connection to a remote cluster — no inbound port required (NAT-friendly).
- Leaf has its **own auth domain** for local clients; binds outbound to a single account on the hub.
- 💡 Use cases: IoT gateways, per-region read replicas, embedding NATS inside a customer VPC.

## Auth Propagation
- Each tier has its own auth scope. JWT/[[NKey]] credentials at the leaf authenticate against the hub.
- Subject namespace is **scoped per account** — leaf clients see only what their bound account exports.
- ⚠️ Permissions don't auto-flow downward; you grant pub/sub rights at each layer explicitly.

## Tags
[[NATS]] [[Edge]] [[Messaging]]
