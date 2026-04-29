# 06 — Networking
🔑 Each container gets a network namespace; drivers (bridge/host/none/overlay) decide how it reaches the world. User-defined bridges add DNS-by-name between containers.

Source: https://docs.docker.com/engine/network/

## Built-in Drivers
| Driver | Use | Notes |
|---|---|---|
| `bridge` | Default for single-host containers | Default `bridge` lacks name-based DNS |
| `host` | Container shares host's net ns | No isolation, no `-p` needed |
| `none` | No networking | Air-gapped jobs |
| `overlay` | Multi-host (Swarm) | VXLAN tunnel between daemons |
| `macvlan` | Container looks like a physical device on LAN | Niche |

## Default vs User-Defined Bridge
- **Default `bridge`** — containers reach each other only by IP; DNS doesn't resolve names.
- **User-defined bridge** — auto-DNS via Docker's resolver at `127.0.0.11`; containers reach each other by **container name** or **service name** ([[Compose]] uses one per project).
```bash
docker network create app
docker run -d --name db   --network app postgres:16
docker run -d --name api  --network app -e PGHOST=db myapi
```

## Port Publishing
- `EXPOSE` is documentation; doesn't open anything.
- `-p HOST:CONTAINER` actually publishes via host iptables/NAT.
```bash
docker run -p 8080:80 nginx          # host:8080 → container:80
docker run -p 127.0.0.1:8080:80 nginx  # bind only loopback
```
- ⚠️ `-p 8080:80` listens on **all** host interfaces by default — bind to `127.0.0.1` for dev.

## Tags
[[Docker]] [[Networking]] [[Compose]]
