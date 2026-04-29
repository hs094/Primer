# 10 — Security
🔑 Containers are not a security boundary by default — harden the image, drop privileges, and constrain the kernel surface.

Source: https://docs.docker.com/engine/security/

## Image Hardening
- **Pin base** by digest, prefer `*-slim` / distroless / scratch.
- **`USER` non-root** — last instruction in Dockerfile sets the runtime UID.
```dockerfile
RUN useradd -u 10001 -r app
USER 10001
```
- Multi-stage to leave compilers/package managers out of the runtime.
- ⚠️ `latest` tag + curl-pipe-bash inside `RUN` = unreviewable supply-chain risk.

## Runtime Flags
| Flag | Effect |
|---|---|
| `--read-only` | Root FS read-only; combine with `--tmpfs /tmp` |
| `--cap-drop=ALL --cap-add=…` | Strip Linux capabilities to the minimum |
| `--security-opt=no-new-privileges:true` | Block setuid escalation |
| `--user 10001:10001` | Run as non-root even if image forgot |
| `--pids-limit`, `--memory`, `--cpus` | Resource caps via cgroups |
| `--security-opt seccomp=profile.json` | Custom syscall allowlist |
| `--security-opt apparmor=profile` | LSM policy (Ubuntu/Debian) |

## Don't
- ⚠️ `--privileged` — disables nearly all isolation; equivalent to root on host.
- ⚠️ Mounting `/var/run/docker.sock` into containers (CI/agents) = container escape primitive.
- ⚠️ Build-time `ARG` for secrets — they end up in `docker history`. Use BuildKit `--mount=type=secret`.

## Secrets at Runtime
- Compose / Swarm `secrets:` mount as files under `/run/secrets`.
- Or inject via env from an external secret manager (Vault, AWS SM) at start.

## Scanning
- **Trivy** — `trivy image myapp:tag` → CVEs, misconfig, secrets, SBOM.
- **grype** — Anchore alternative; pairs with `syft` for SBOM.
- 💡 Wire into CI; fail builds on `HIGH`/`CRITICAL` with a known-fixed version.

## Tags
[[Docker]] [[Security]] [[Containers]] [[Supply-Chain]]
