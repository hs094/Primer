# 13 — Helm

🔑 Helm packages a set of Kubernetes manifests + Go templates + values into a versioned, installable, upgradable, rollback-able release.

Source: https://helm.sh/docs/topics/charts/

## Chart Layout
```
mychart/
├── Chart.yaml         # name, version, appVersion, dependencies
├── values.yaml        # default config (overridable)
├── values.schema.json # optional JSON Schema for values
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── _helpers.tpl   # named template snippets
│   └── NOTES.txt
├── charts/            # bundled subcharts
└── crds/              # CRDs (installed before templates, not templated)
```

## Templating
- **Engine:** Go templates + ~50 [Sprig](https://masterminds.github.io/sprig/) functions.
- **Built-ins:** `.Values`, `.Release.Name`, `.Release.Namespace`, `.Chart`, `.Capabilities`, `.Files`.
- **Helpers:** `{{ include "mychart.fullname" . }}` from `_helpers.tpl`.

```yaml
spec:
  replicas: {{ .Values.replicas | default 2 }}
  template:
    spec:
      containers:
        - name: app
          image: "{{ .Values.image.repo }}:{{ .Values.image.tag }}"
```

## Lifecycle Commands
| Command | Effect |
|---|---|
| `helm install <rel> <chart> -f values.yaml` | Create release |
| `helm upgrade --install <rel> <chart>` | Idempotent install/upgrade |
| `helm rollback <rel> <revision>` | Roll back to a prior revision |
| `helm history <rel>` | Show release revisions |
| `helm template <chart>` | Render to YAML locally (no cluster) |
| `helm lint` | Static checks |

## Dependencies
Declared in `Chart.yaml`:
```yaml
dependencies:
  - name: postgresql
    version: "15.x.x"
    repository: "oci://registry-1.docker.io/bitnamicharts"
```
`helm dependency update` pulls them into `charts/`.

## OCI Registries
Charts can be pushed to any OCI registry (Docker Hub, GHCR, ECR, GAR):
```bash
helm push mychart-1.0.0.tgz oci://ghcr.io/me/charts
helm install rel oci://ghcr.io/me/charts/mychart --version 1.0.0
```

## ⚠️ Gotchas
- CRDs in `crds/` are installed **once** and **never** upgraded by Helm — manage their lifecycle separately.
- Release state lives in cluster Secrets (`sh.helm.release.v1.<rel>.<rev>`); don't lose them.
- 💡 `--atomic` rolls back automatically if upgrade fails.

## Tags
[[Kubernetes]] [[Helm]] [[Chart]]
