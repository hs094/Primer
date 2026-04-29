# 13 — Helm

🔑 Helm packages manifests + Go templates + values into a versioned, installable, upgradable, rollback-able release.

Source: https://helm.sh/docs/topics/charts/

## Chart Layout
```
mychart/
├── Chart.yaml          # name, version, appVersion, deps
├── values.yaml         # default config
├── values.schema.json  # optional JSON Schema
├── templates/          # Go-templated manifests + _helpers.tpl + NOTES.txt
├── charts/             # bundled subcharts
└── crds/               # CRDs — installed once, never upgraded
```

## Templating
Go templates + ~50 Sprig funcs. Built-ins: `.Values`, `.Release.{Name,Namespace}`, `.Chart`, `.Capabilities`, `.Files`. Reuse via `{{ include "mychart.fullname" . }}` helpers.

```yaml
spec:
  replicas: {{ .Values.replicas | default 2 }}
  template:
    spec:
      containers:
        - name: app
          image: "{{ .Values.image.repo }}:{{ .Values.image.tag }}"
```

## Lifecycle
| Command | Effect |
|---|---|
| `helm install <rel> <chart> -f values.yaml` | Create release |
| `helm upgrade --install <rel> <chart>` | Idempotent upsert |
| `helm rollback <rel> <revision>` | Revert to prior revision |
| `helm history <rel>` | Show revisions |
| `helm template <chart>` | Render locally, no cluster |
| `helm lint` / `helm diff upgrade` | Static / preview checks |

## Dependencies & OCI
`Chart.yaml` declares `dependencies` (name, version, repository); `helm dependency update` pulls them into `charts/`. Charts publish to any OCI registry (`helm push mychart-1.0.0.tgz oci://ghcr.io/me/charts`).

## ⚠️ Gotchas
- CRDs in `crds/` are installed once and **never upgraded** — manage them out of band.
- Release state lives in cluster Secrets (`sh.helm.release.v1.<rel>.<rev>`); don't lose them.
- 💡 `--atomic` auto-rolls-back a failed upgrade.

## Tags
[[Kubernetes]] [[Helm]] [[Chart]]
