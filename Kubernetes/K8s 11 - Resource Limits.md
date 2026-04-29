# 11 — Resource Limits

🔑 Requests are what the scheduler reserves; limits are what the kernel enforces. Memory over limit = OOMKilled; CPU over limit = throttled.

Source: https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/

## Requests vs Limits
| | CPU | Memory |
|---|---|---|
| **Request** | Scheduling guarantee; CFS shares | Guaranteed bytes; node admission |
| **Limit** | CFS quota → throttling | Hard cap → OOMKill on overflow |

Units: CPU in cores or millicores (`500m` = 0.5). Memory in `Ki/Mi/Gi` (binary, not decimal).

## QoS Classes (assigned by kubelet, not user)
| Class | Rule | Eviction Order |
|---|---|---|
| **Guaranteed** | Every container has request == limit (cpu + mem) | Last to evict |
| **Burstable** | Has at least one request, but not Guaranteed | Middle |
| **BestEffort** | No requests or limits | First to evict |

## Example
```yaml
resources:
  requests: { cpu: "200m", memory: "256Mi" }
  limits:   { cpu: "1",    memory: "512Mi" }
```

## OOMKilled
- Container's memory cgroup hits limit → kernel OOM killer → container restarted.
- Visible as `lastState.terminated.reason: OOMKilled`.
- Memory has **no throttling** — you cannot "burst" past the limit.

## Namespace Guardrails
- **ResourceQuota** — caps aggregate `requests.cpu`, `limits.memory`, object counts per namespace.
- **LimitRange** — sets default + min/max per container, so Pods without requests still get sane values.

```yaml
apiVersion: v1
kind: LimitRange
metadata: { name: defaults, namespace: dev }
spec:
  limits:
    - type: Container
      default:        { cpu: 500m, memory: 512Mi }
      defaultRequest: { cpu: 100m, memory: 128Mi }
```

## ⚠️ Gotchas
- Setting **only** a CPU limit (no request) is a common cause of mysterious throttling — the request defaults to the limit, oversizing the reservation.
- Java/Go runtimes pre-1.18 / pre-Java 10 didn't see cgroup limits → set `GOMAXPROCS` / `-XX:+UseContainerSupport`.
- 💡 Avoid CPU limits in latency-sensitive paths; rely on requests + HPA (see [[K8s 12 - Autoscaling]]).

## Tags
[[Kubernetes]] [[QoS]] [[ResourceQuota]] [[LimitRange]]
