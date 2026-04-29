# 12 — Autoscaling

🔑 Three axes: HPA scales replicas, VPA scales per-Pod resources, Cluster Autoscaler / Karpenter scale nodes. KEDA hooks HPA up to event sources.

Source: https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/

## Horizontal Pod Autoscaler (HPA)
Scales replica count of a [[Deployment]] / StatefulSet on metrics from `metrics.k8s.io` (CPU/mem) or `custom.metrics.k8s.io` / `external.metrics.k8s.io`.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata: { name: api }
spec:
  scaleTargetRef: { apiVersion: apps/v1, kind: Deployment, name: api }
  minReplicas: 2
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target: { type: Utilization, averageUtilization: 70 }
  behavior:
    scaleDown: { stabilizationWindowSeconds: 300 }
```
Formula: `desired = ceil(current * currentMetric / targetMetric)`. Requires **metrics-server**.

## Vertical Pod Autoscaler (VPA)
Right-sizes `requests`/`limits` based on observed usage. Modes:
- `Off` — recommendations only (`kubectl describe vpa`).
- `Initial` — apply on Pod create.
- `Auto` — evict + recreate Pods to apply (disruptive!).

⚠️ Don't run VPA `Auto` together with HPA on **CPU/mem** — they fight. Use HPA on custom metrics + VPA on resources.

## Cluster Autoscaler
Scales node count up when Pods are unschedulable, down when nodes are underutilized. Works against a node group / ASG / MIG. Slow (minutes), conservative.

## Karpenter
Modern alternative (AWS, then upstream). Provisions right-sized nodes directly from instance type catalog instead of fixed node groups; faster and bin-packs better.

## KEDA
Event-driven autoscaling: scale on Kafka lag, RabbitMQ queue depth, Redis list length, SQS, cron, Prometheus query. Creates an HPA under the hood and can scale to zero.

## ⚠️ Gotchas
- HPA needs `resources.requests` set on Pods to compute Utilization%.
- 💡 `behavior.scaleDown.stabilizationWindowSeconds` prevents flapping; tune it for spiky workloads.

## Tags
[[Kubernetes]] [[HPA]] [[VPA]] [[KEDA]] [[Karpenter]]
