# 03 — Workloads

🔑 Pick the controller by what your app needs: identity, ordering, schedule, or none of the above.

Source: https://kubernetes.io/docs/concepts/workloads/controllers/

## The Five
| Controller | Use Case | Identity | Ordering |
|---|---|---|---|
| **Deployment** | Stateless apps, rolling updates | Interchangeable | None |
| **StatefulSet** | DBs, brokers (PG, Kafka, ES) | Stable name + DNS | Ordered create/delete |
| **DaemonSet** | One Pod per node (log agent, CNI) | Per node | N/A |
| **Job** | Run-to-completion batch | N/A | Parallelism configurable |
| **CronJob** | Scheduled Job | N/A | Cron expression |

## Deployment Mechanics
- Owns a ReplicaSet; ReplicaSet owns Pods.
- **Rolling update** via `maxSurge` / `maxUnavailable`.
- `kubectl rollout status / undo / history deployment/<n>`.

## StatefulSet Mechanics
- Pods named `<sts>-0`, `<sts>-1`, … with stable DNS via headless [[Service]].
- Per-Pod PVC from `volumeClaimTemplates` — survives rescheduling.
- Scale up `0 → N`; scale down `N → 0`. Updates roll in reverse ordinal order.

## DaemonSet
- One Pod on every (matching) node. Tolerates control-plane taints if needed.
- Ideal for `node-exporter`, fluent-bit, CSI node plugin, CNI agent.

## Job / CronJob
- `completions` + `parallelism` for batch fanout.
- CronJob fires `Job` per schedule; set `concurrencyPolicy: Forbid` to skip overlap.

## ⚠️ Gotchas
- StatefulSet PVCs are **not** deleted on scale-down by default — set `persistentVolumeClaimRetentionPolicy`.
- `Deployment` rolling update requires probes to be honest (see [[K8s 10 - Probes and Health]]).
- 💡 Use `kubectl rollout restart deploy/<n>` to force a fresh roll without changing the spec.

## Tags
[[Kubernetes]] [[Deployment]] [[StatefulSet]] [[DaemonSet]] [[Job]]
