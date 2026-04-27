# PD 01 — Production Overview

🔑 **Production Temporal = your app code (Workflows, Activities, [[Temporal EN 06 - Workers|Workers]]) + a production-ready [[Temporal EN 11 - Temporal Service|Temporal Service]] (Cloud or self-hosted).**

## Two halves of a deployment

The Temporal Platform splits cleanly into:

1. **Application code** — Workflows, Activities, and [[Temporal DV 15 - Worker Processes|Worker Processes]] that run on your own infrastructure.
2. **Temporal Service** — orchestrates Workflow Execution, persists [[Temporal EN 07 - Event History|Event History]], routes Tasks, and surfaces the Web UI.

You always own #1. For #2 you pick managed Cloud or self-host.

## Cloud vs self-hosted

| Concern | Temporal Cloud | Self-hosted |
|---|---|---|
| Service ops | Anthropic-managed | You run it ([[Temporal SH 01 - Self-Hosted Overview|SH 01]]) |
| Scaling shards/persistence | Handled | You tune it |
| Namespace mTLS / certs | Provisioned via tcld/UI | You manage CA |
| Codec server | You host | You host |
| Workers | **Always your infra** | **Always your infra** |

> Workers never run in Cloud. They live where your code and external dependencies live.

## Local dev does not count

```bash
temporal server start-dev
```

Single-process, in-memory persistence, fine for laptop iteration. **Not a production target.** It is not durable across restarts and lacks the multi-role topology of a real cluster.

⚠️ "It works in `start-dev`" tells you nothing about whether shards, persistence, and Worker pools will hold up under load.

## Production checklist (the short list)

- [ ] **Persistence** — Cassandra / PostgreSQL / MySQL sized for shard count and retention.
- [ ] **Shards** — set once at namespace creation; cannot be changed without a migration.
- [ ] **TLS** — mTLS between SDK clients, Workers, and Frontend; internode TLS for self-hosted ([[Temporal TS 03 - Last Connection Error|TS 03]] for cert expiry footgun).
- [ ] **Codec server** — if any payload is sensitive, deploy one before users hit prod ([[Temporal PD 03 - Codecs and Encryption|PD 03]]).
- [ ] **Worker capacity** — provision pollers, slots, and replicas per Task Queue ([[Temporal BP 02 - Worker Performance|BP 02]]).
- [ ] **Worker Versioning** — opt in from day one ([[Temporal PD 02 - Worker Deployments|PD 02]]).
- [ ] **Monitoring** — scrape SDK + Service metrics, alert on schedule-to-start latency and resource_exhausted ([[Temporal SH 08 - Monitoring|SH 08]]).
- [ ] **Retention period** — set per Namespace; closed Workflows visible until it elapses.
- [ ] **Blob/payload review** — confirm no Workflow input or Activity result hugs the 2 MB / 4 MB ceiling ([[Temporal TS 01 - Blob Size Limit Error|TS 01]]).

## Capacity, briefly

- **Frontend Service** scales horizontally, gates RPS via `RpsLimit` / `ConcurrentLimit`.
- **History Service** is shard-bound; one shard = one owner at a time.
- **Matching Service** scales with Task Queue partitions.
- **Worker Service** is internal (replication / archival), distinct from your application Workers.

⚠️ Shard count is the one decision you cannot easily walk back. Pick high enough for projected load; you cannot resize without a full data migration.

## Security boundaries

- **In transit** — TLS everywhere (frontend, internode, persistence).
- **At rest** — disk encryption is on you (Cloud handles this; self-host = your problem).
- **Payload** — only a [[Temporal DV 21 - Data Handling|Codec]] keeps Workflow inputs/results encrypted server-side. TLS does not.

## Deployment topology you actually ship

```
[ SDK Clients ] ─┐
                 ├─► [ Frontend (mTLS) ] ─► [ History | Matching | Worker | Persistence ]
[ Workers     ] ─┘                                   ▲
                                                     │
                                          [ Codec Server (your infra) ]
```

Workers, clients, and the codec server all live in your VPC. Only the Frontend is the public-facing surface, and only via mTLS.

## Common production mistakes

⚠️ Treating dev-server config as a baseline.
⚠️ Sizing Workers from CPU alone, ignoring poller/slot saturation ([[Temporal TS 04 - Performance Bottlenecks|TS 04]]).
⚠️ Letting Namespace mTLS certs expire ([[Temporal TS 03 - Last Connection Error|TS 03]]).
⚠️ Skipping Worker Versioning, then trying to retrofit `patched()` everywhere ([[Temporal PD 02 - Worker Deployments|PD 02]]).
⚠️ Putting raw PII into Workflow inputs without a codec ([[Temporal PD 03 - Codecs and Encryption|PD 03]]).

## Failure modes you should already have answers for

- A Worker fleet replica dies — does another pick up the Task? (yes, if Task Queue is shared)
- Frontend rejects RPCs with `ResourceExhausted` — what's your dashboard? ([[Temporal TS 02 - Deadline Exceeded Error|TS 02]])
- A Workflow Task exceeds 4 MB — is it terminated or recoverable? ([[Temporal TS 01 - Blob Size Limit Error|TS 01]])
- Persistence latency spikes — do you alert on `temporal_request_latency` before users notice?

💡 **Takeaway:** You own the Workers and the codec; Cloud or self-host owns the Service. Plan capacity, TLS, versioning, and codecs before the first prod Workflow ever starts.
