# 02 — Pods

🔑 A Pod is a co-scheduled group of containers sharing a network namespace, IPC, and volumes — the smallest thing the scheduler places.

Source: https://kubernetes.io/docs/concepts/workloads/pods/

## What's Shared
| Resource | Behavior |
|---|---|
| **Network namespace** | One IP; containers reach each other on `localhost:<port>`. |
| **IPC + UTS** | Shared by default. |
| **Volumes** | Mounted into any container that lists them. |
| **Filesystem** | Each container has its own rootfs from its image. |

## Multi-Container Patterns
- **Init containers** — run to completion sequentially before app containers start (migrations, fetch config, wait-for-dep).
- **Sidecars** — long-running helpers next to the main app (log shipper, proxy, mesh data plane). 1.29+ models them as restartable init containers.
- **Ephemeral containers** — injected at runtime via `kubectl debug`, never restart.

## Lifecycle Phases
`Pending` → `Running` → `Succeeded` | `Failed` | `Unknown`. The Pod object is immutable once scheduled (besides image, tolerations, resources in 1.27+).

## Restart Policy (`spec.restartPolicy`)
- `Always` (default for [[Deployment]]) — restart on any exit.
- `OnFailure` — restart only on non-zero exit (Jobs).
- `Never` — let it die.

## ⚠️ Gotchas
- Pods are **ephemeral** — IPs change, names change. Always front them with a [[Service]].
- Don't `kubectl run` raw Pods in prod; use a workload controller so they reschedule.
- 💡 Pod `terminationGracePeriodSeconds` defaults to 30s — bump it for slow drainers (see [[K8s 10 - Probes and Health]]).

## 🧪 Try It
```bash
kubectl run tmp --image=busybox --rm -it --restart=Never -- sh
kubectl get pod tmp -o yaml | grep -A2 restartPolicy
```

## Tags
[[Kubernetes]] [[Pod]] [[Sidecar]] [[Init Container]]
