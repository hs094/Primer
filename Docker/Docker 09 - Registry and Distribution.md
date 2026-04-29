# 09 — Registry and Distribution
🔑 A registry stores [[OCI]] artifacts (images, indexes, signatures) keyed by digest; pick one for trust, latency, and rate-limit reasons.

Source: https://docs.docker.com/docker-hub/

## Common Registries
| Registry | Host | Notes |
|---|---|---|
| Docker Hub | `docker.io` | Default; biggest public catalog; rate-limited for anon |
| GitHub Container Registry | `ghcr.io` | Free for public; GitHub OIDC auth in Actions |
| AWS ECR | `<acct>.dkr.ecr.<region>.amazonaws.com` | IAM-auth; private + public sub-tier |
| Google Artifact Registry / GCR | `*-docker.pkg.dev` | IAM-auth; replaced legacy `gcr.io` |
| Azure ACR | `<name>.azurecr.io` | AAD-auth; geo-replication |
| Self-hosted | `registry:2`, Harbor, Zot | OCI-compliant; full control |

## Pull Rate Limits (Docker Hub)
- Anonymous: 100 pulls / 6h per source IP.
- Authenticated free: 200 pulls / 6h.
- ⚠️ Shared NAT (CI runners, k8s nodes) hits limits fast → log in or mirror.
- 💡 Use a pull-through cache (`registry:2` with `proxy.remoteurl`) or push base images to your own registry.

## Tagging and Digests
```bash
docker tag myapp:dev ghcr.io/acme/myapp:v1.2.3
docker push ghcr.io/acme/myapp:v1.2.3
# Pin in production by digest:
docker pull ghcr.io/acme/myapp@sha256:abc123…
```

## Image Signing
- **cosign** (Sigstore) — keyless signing via OIDC, verification in admission controllers.
- `cosign sign $IMG` / `cosign verify --certificate-identity-regexp … $IMG`.
- 💡 Pair with SBOMs (`syft`) and attestations for supply-chain integrity.

## Tags
[[Docker]] [[Registry]] [[OCI]] [[Supply-Chain]]
