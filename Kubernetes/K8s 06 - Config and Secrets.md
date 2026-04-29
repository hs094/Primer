# 06 — Config and Secrets

🔑 ConfigMap = plain config, Secret = sensitive config (base64-encoded, **not encrypted by default**). Both surface as env vars or files.

Source: https://kubernetes.io/docs/concepts/configuration/configmap/

## ConfigMap vs Secret
| | ConfigMap | Secret |
|---|---|---|
| Sensitive? | No | Yes |
| Encoding | Plain | Base64 |
| Encrypted at rest | No (unless EncryptionConfig) | Same — opt in via KMS |
| Size limit | 1 MiB | 1 MiB |

## Consumption
**As env vars (one-by-one):**
```yaml
env:
  - name: DB_HOST
    valueFrom: { configMapKeyRef: { name: app-cfg, key: db_host } }
```
**Bulk import every key:**
```yaml
envFrom:
  - configMapRef: { name: app-cfg }
  - secretRef:    { name: app-secrets }
```
**As files (auto-updates on change, ~minute lag):**
```yaml
volumes:
  - name: cfg
    configMap: { name: app-cfg }
volumeMounts:
  - { name: cfg, mountPath: /etc/app, readOnly: true }
```

## Encryption at Rest
Pass `--encryption-provider-config` to kube-apiserver. Use **KMS v2** in production (envelope encryption against AWS KMS, GCP KMS, Vault Transit, …).

## External Secret Operators
- **External Secrets Operator (ESO)** — syncs from AWS Secrets Manager, GCP SM, Vault, Azure KV → native [[Secret]].
- **HashiCorp Vault** + Vault Agent Injector — sidecar writes secrets onto a tmpfs.
- **SOPS + Flux/Argo** — git-encrypted secrets, decrypted in-cluster.

## ⚠️ Gotchas
- **Base64 ≠ encryption.** `kubectl get secret -o yaml` exposes everything to anyone with `get secret`.
- env-var injection is **fixed at Pod start** — file mounts pick up edits, env vars do not.
- 💡 Lock down `secrets` with [[RBAC]] (see [[K8s 09 - RBAC]]); consider per-namespace separation.

## Tags
[[Kubernetes]] [[ConfigMap]] [[Secret]] [[Vault]]
