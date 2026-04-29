# 15 — Operators and CRDs

🔑 An Operator is a custom controller that turns a CRD ("MyDB") into running infrastructure via a reconcile loop — SRE knowledge as code.

Source: https://kubernetes.io/docs/concepts/extend-kubernetes/operator/

## CustomResourceDefinition (CRD)
Extends the Kubernetes API with new types, validated by an OpenAPI schema.

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata: { name: sampledbs.databases.example.com }
spec:
  group: databases.example.com
  scope: Namespaced
  names: { kind: SampleDB, plural: sampledbs }
  versions:
    - { name: v1, served: true, storage: true,
        schema: { openAPIV3Schema: { type: object } } }
```
A user-facing CR is then `kind: SampleDB` with a `spec` like `{ version: "16", storage: 50Gi }`.

## Reconcile Loop
```
forever:
  observe = read CR + actual cluster state
  plan    = diff(desired, observed)
  act     = create/update/delete children (STS, PVC, Service, ...)
  status  = write back to CR.status
```
Idempotent; re-runs on every watch event or periodic resync.

## Frameworks
| Framework | Lang |
|---|---|
| **kubebuilder** / **Operator SDK** | Go (controller-runtime) |
| **Kopf** | Python |
| **Java Operator SDK** | Java |
| **kube-rs** | Rust |

## Real Operators
- **Strimzi** — Kafka clusters, topics, users.
- **CloudNativePG** — Postgres with HA, backups, failover.
- **Prometheus Operator** — `ServiceMonitor`, `PrometheusRule`.
- **cert-manager** — ACME `Certificate` automation.
- **Argo CD** / **Flux** — GitOps controllers.

## ⚠️ Gotchas
- A CRD without a controller is just a glorified ConfigMap — nothing reconciles it.
- `status` is controller-owned; use the `/status` subresource so users can't write it.
- 💡 Check OperatorHub.io before writing your own.

## Tags
[[Kubernetes]] [[CRD]] [[Operator]] [[Reconcile]]
