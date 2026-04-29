# 10 — Probes and Health

🔑 Liveness restarts a sick container, readiness pulls it out of [[Service]] rotation, startup gates the other two for slow boots.

Source: https://kubernetes.io/docs/concepts/configuration/liveness-readiness-startup-probes/

## Three Probes
| Probe | Failure → | Use For |
|---|---|---|
| **livenessProbe** | Restart container | Detecting deadlocks |
| **readinessProbe** | Remove from Service endpoints | Warmup, dependency checks |
| **startupProbe** | Kill container; disables liveness/readiness until it passes once | Slow JVMs, large model loads |

## Probe Mechanisms
- **httpGet** — 2xx/3xx is success.
- **tcpSocket** — TCP connect succeeds.
- **exec** — command exits 0.
- **grpc** — `grpc.health.v1.Health/Check` returns SERVING.

## Example
```yaml
startupProbe:
  httpGet: { path: /healthz, port: 8080 }
  failureThreshold: 30   # 30 * 10s = 5min boot budget
  periodSeconds: 10
livenessProbe:
  httpGet: { path: /healthz, port: 8080 }
  periodSeconds: 10
  failureThreshold: 3
readinessProbe:
  httpGet: { path: /ready, port: 8080 }
  periodSeconds: 5
```

## Graceful Shutdown
```yaml
terminationGracePeriodSeconds: 60
lifecycle:
  preStop:
    exec: { command: ["/bin/sh","-c","sleep 10"] }   # let LB notice we're gone
```
Sequence: delete → endpoint removed + `preStop` runs → `SIGTERM` → up to grace period → `SIGKILL`.

## ⚠️ Gotchas
- Use `startupProbe` instead of bumping `initialDelaySeconds` on liveness — the latter just delays *every* restart.
- A failing liveness probe under load creates a death spiral; consider higher `failureThreshold` and check on a cheap endpoint.
- Readiness probe failure does **not** restart the container. Liveness will.
- 💡 `/healthz` should be cheap and not check downstream deps; `/ready` can.

## Tags
[[Kubernetes]] [[Probe]] [[Pod]]
