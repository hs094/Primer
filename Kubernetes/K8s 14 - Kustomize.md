# 14 — Kustomize

🔑 Template-free customization via overlays — a base of plain YAML, patched per environment. Built into `kubectl -k`.

Source: https://kustomize.io/

## Layout
`base/` holds the canonical manifests + a `kustomization.yaml`; `overlays/{dev,prod}/` each have their own `kustomization.yaml` referencing the base plus environment-specific patches.

## Overlay Example
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: prod
resources: [../../base]
images:
  - { name: api, newTag: v1.4.2 }
replicas:
  - { name: api, count: 5 }
patches:
  - path: resources-patch.yaml
    target: { kind: Deployment, name: api }
```

## Patches
- **Strategic merge** — YAML fragment, Kubernetes-aware merge rules.
- **JSON 6902** — RFC-6902 ops (`add`/`replace`/`remove`) with JSON Pointer paths.

## Generators
`configMapGenerator` / `secretGenerator` build ConfigMap/Secret from files or literals; output gets a content-hash suffix so changes trigger rolling updates (refs are rewritten).

## Apply
```bash
kubectl apply -k overlays/prod
kubectl kustomize overlays/prod   # render only
kubectl diff -k overlays/prod     # preview
```

## Helm vs Kustomize
| | Helm | Kustomize |
|---|---|---|
| Style | Go templates | Patches / overlays |
| Distribution | OCI charts | Just git |
| Logic | Yes | No (declarative) |
| State | Release secrets | None |

Hybrid (`helm template … | kustomize build`) is common.

## ⚠️ Gotchas
- No conditionals / loops by design — reach for [[Helm]] if needed.
- 💡 Generator hash names break hard-coded refs; always reference by base `name`.

## Tags
[[Kubernetes]] [[Kustomize]] [[Helm]]
