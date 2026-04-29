# 16 — Observability and Debug

🔑 Start with `describe` + events, then `logs`, then `exec` / `debug` containers. Metrics-server gives you `kubectl top`; Prometheus + Grafana for the real stack.

Source: https://kubernetes.io/docs/tasks/debug/debug-application/debug-running-pod/

## kubectl Triage Toolkit
| Command | When |
|---|---|
| `kubectl get pods -o wide` | Quick state, node, IP. |
| `kubectl describe pod <p>` | Events, conditions, image pull errors, scheduling failures. |
| `kubectl logs <p> [-c <c>] [--previous]` | App stdout/stderr; `--previous` for last crash. |
| `kubectl exec -it <p> -- sh` | Live shell in the container. |
| `kubectl port-forward <p> 8080:80` | Local TCP tunnel for ad-hoc curl. |
| `kubectl events -n <ns> --watch` | Stream cluster events. |
| `kubectl top pod / node` | Live CPU/mem (needs metrics-server). |

## Common Diagnoses (from `describe` events)
| Symptom | Likely Cause |
|---|---|
| `ImagePullBackOff` | Bad tag, missing pull secret. |
| `CrashLoopBackOff` | App crashes on start; check `logs --previous`. |
| `Pending` (FailedScheduling) | Insufficient resources, taint/tolerance, PVC pending. |
| `OOMKilled` | Memory limit too low (see [[K8s 11 - Resource Limits]]). |
| `Unhealthy: liveness probe failed` | Bad probe or slow startup. |

## Ephemeral Debug Containers
Inject a sidecar **into a running Pod** without modifying the spec — perfect for distroless images:
```bash
kubectl debug -it mypod --image=busybox:1.36 --target=app
kubectl debug node/ip-10-0-1-5 -it --image=ubuntu  # node shell
kubectl debug mypod --copy-to=mypod-debug --set-image=app=busybox  # cloned Pod
```

## Metrics & Logs Stack
- **metrics-server** — short-term resource metrics for HPA + `kubectl top`.
- **kube-prometheus-stack** (Helm) — Prometheus Operator + Grafana + node-exporter + kube-state-metrics.
- **Loki / Elastic / OpenSearch** — log aggregation.
- **OpenTelemetry Collector** — traces + metrics + logs pipeline.

## ⚠️ Gotchas
- `kubectl logs` only shows what the container wrote to stdout/stderr — file logs need a sidecar or DaemonSet collector.
- Events are **garbage-collected** (~1h). Capture them quickly or ship to a backend.
- 💡 `kubectl debug` requires `EphemeralContainers` enabled (default since 1.25).

## Tags
[[Kubernetes]] [[kubectl]] [[Prometheus]] [[Observability]]
