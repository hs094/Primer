# 15 — Operators and CRDs

🔑 An Operator is a custom controller that turns a CRD ("MyDB") into running infrastructure by reconciling desired state into reality — codifying SRE knowledge.

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
  names: { kind: SampleDB, plural: sampledbs, singular: sampledb }
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                version: { type: string }
                storage: { type: string }
```

## Custom Resource (CR)
```yaml
apiVersion: databases.example.com/v1
kind: SampleDB
metadata: { name: orders-db }
spec: { version: "16", storage: 50Gi }
```

## Reconcile Loop
```
forever:
  observe   = read CR + actual cluster state
  plan      = diff(desired, observed)
  act       = create/update/delete child objects (StatefulSet, PVC, Service, ...)
  status    = write back to CR.status
```
Idempotent. Re-runs cheaply on every event or periodic resync.

## Frameworks
| Framework | Lang |
|---|---|
| **kubebuilder** / **Operator SDK** | Go (controller-runtime) |
| **Kopf** | Python |
| **Java Operator SDK** | Java |
| **kube-rs** | Rust |

## Real Operators
- **Strimzi** — Kafka clusters, topics, users.
- **CloudNativePG** — Postgres with backups, failover, replicas.
- **Prometheus Operator** — `Prometheus`, `ServiceMonitor`, `PrometheusRule` CRDs.
- **cert-manager** — `Certificate`, `Issuer`, ACME automation.
- **Argo CD / Argo Rollouts**, **Flux** — GitOps controllers.

## ⚠️ Gotchas
- A CRD without a controller is just a glorified ConfigMap — nothing reconciles it.
- `status` should only be written by the controller; spec by users. Use the `/status` subresource.
- 💡 OperatorHub.io catalogs hundreds of off-the-shelf operators — check before building one.

## Tags
[[Kubernetes]] [[CRD]] [[Operator]] [[Reconcile]]
