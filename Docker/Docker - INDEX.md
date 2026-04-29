# Docker — INDEX

🔑 Containers are isolated processes packaged with their dependencies — Docker is the toolchain (build, ship, run) on top of [[OCI]] standards and Linux kernel primitives.

Source: https://docs.docker.com/

## Notes
| # | Note | Focus |
|---|---|---|
| 01 | [[Docker 01 - Overview]] | Containers, namespaces+cgroups, daemon vs rootless |
| 02 | [[Docker 02 - Images and Layers]] | Layered FS, overlayfs, digests, cache ordering |
| 03 | [[Docker 03 - Dockerfile]] | Instruction reference + `.dockerignore` |
| 04 | [[Docker 04 - Multi-Stage Builds]] | `COPY --from`, distroless, Alpine |
| 05 | [[Docker 05 - BuildKit]] | Parallel builds, cache/secret mounts, `buildx` |
| 06 | [[Docker 06 - Networking]] | Bridge, host, none, overlay, port publishing |
| 07 | [[Docker 07 - Volumes and Bind Mounts]] | Named volumes, binds, tmpfs, backups |
| 08 | [[Docker 08 - Compose]] | `docker-compose.yml`, healthchecks, profiles |
| 09 | [[Docker 09 - Registry and Distribution]] | Hub, GHCR, ECR/GCR/ACR, cosign, rate limits |
| 10 | [[Docker 10 - Security]] | Non-root, caps, seccomp, scanning |

## Mental Model
- **Build** → Dockerfile + [[BuildKit]] produces an image (layers + manifest).
- **Ship** → push image to a registry (Hub, GHCR, ECR…).
- **Run** → daemon (or rootless) launches a container = process in namespaces + cgroups + overlayfs rootfs.
- **Compose** → declarative multi-container apps for dev/CI; [[Kubernetes]] for prod orchestration.

## Read Order
1 → 2 → 3 → 4 → 5 (build path), then 6 → 7 → 8 (run/ops), then 9 → 10 (distribution + hardening).

## Tags
[[Docker]] [[Containers]] [[OCI]] [[BuildKit]] [[Compose]]
