# 01 — Overview
🔑 A container is a host process isolated by Linux kernel namespaces and limited by cgroups, with a rootfs assembled from image layers — Docker is the toolchain that builds, ships, and runs them.

Source: https://docs.docker.com/get-started/docker-overview/

## Container = Process + Isolation + Limits
| Mechanism | What it isolates / does |
|---|---|
| **Namespaces** | PID, mount, net, UTS, IPC, user — each container sees its own slice |
| **cgroups** | CPU, memory, I/O, PIDs — prevents resource exhaustion |
| **Capabilities** | Fine-grained root powers; default drops most |
| **overlayfs** | Stacks read-only image layers + a writable container layer |

## Image vs Container
- **Image** — immutable, layered template (read-only) addressed by digest.
- **Container** — running instance: image layers + thin writable layer; ephemeral unless state goes to a volume.

## OCI Standard
- **OCI Image Spec** — manifest + config + layer tarballs (what's in a registry).
- **OCI Runtime Spec** — how a runtime (e.g. `runc`, `crun`) launches a container from a bundle.
- 💡 Docker, Podman, containerd, CRI-O all consume OCI artifacts → images are portable.

## Daemon vs Rootless
- **Daemon mode** — `dockerd` runs as root, clients talk to `/var/run/docker.sock`. Powerful, but socket = root.
- **Rootless mode** — daemon runs as your user via user namespaces; safer on shared hosts.
- ⚠️ Mounting the docker socket into a container ≈ giving it root on the host.

## Tags
[[Docker]] [[Containers]] [[OCI]] [[Linux]] [[Namespaces]]
