# 14 — Kustomize

🔑 Template-free config customization via overlays — start with a base of plain YAML, then patch it per environment. Built into `kubectl -k`.

Source: https://kustomize.io/

## Layout
```
manifests/
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml
└── overlays/
    ├── dev/
    │   ├── kustomization.yaml
    │   └── replica-patch.yaml
    └── prod/
        ├── kustomization.yaml
        └── resources-patch.yaml
```

## Base `kustomization.yaml`
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
commonLabels: { app: api }
```

## Overlay
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: prod
resources:
  - ../../base
images:
  - name: api
    newTag: v1.4.2
replicas:
  - name: api
    count: 5
patches:
  - path: resources-patch.yaml
    target: { kind: Deployment, name: api }
```

## Patch Types
- **Strategic merge** — YAML fragment merged by Kubernetes-aware rules.
- **JSON 6902** — RFC-6902 ops (`add`, `replace`, `remove`) with JSON Pointer paths.

## Generators
- `configMapGenerator` / `secretGenerator` — build ConfigMap/Secret from files or literals; auto-suffixed hash so changes trigger rolling updates.

```yaml
configMapGenerator:
  - name: app-cfg
    files: [config/app.yaml]
```

## Apply
```bash
kubectl apply -k overlays/prod
kubectl kustomize overlays/prod   # render only
```

## Helm vs Kustomize
| | Helm | Kustomize |
|---|---|---|
| Style | Templates | Patches/overlays |
| Distribution | Charts (versioned, OCI) | Just git |
| Logic in templates | Yes (Go) | No (declarative only) |
| State | Release secrets | None — pure YAML |
| Hybrid | `helm template ... | kustomize` is common |

## ⚠️ Gotchas
- No conditionals / loops by design — if you need them, reach for [[Helm]].
- Generator hashes change Resource names; reference via `name` (Kustomize rewrites refs).
- 💡 `kubectl diff -k overlays/prod` before apply.

## Tags
[[Kubernetes]] [[Kustomize]] [[Helm]]
