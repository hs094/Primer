# 07 — Volumes and Bind Mounts
🔑 Container writable layer is ephemeral — persistent state lives in volumes (Docker-managed), bind mounts (host paths), or tmpfs (RAM).

Source: https://docs.docker.com/engine/storage/volumes/

## Three Mount Types
| Type | Where data lives | Use |
|---|---|---|
| **Named volume** | `/var/lib/docker/volumes/<name>` (Docker-managed) | DB data, app state — preferred default |
| **Bind mount** | Any host path | Source code in dev, host configs |
| **tmpfs** | Host RAM only | Secrets in flight, scratch, perf |

## Syntax
```bash
# Named volume
docker run -v pgdata:/var/lib/postgresql/data postgres:16

# Bind mount (source code in dev)
docker run -v $(pwd):/app node:20 npm test

# Modern --mount (clearer, supports more options)
docker run --mount type=volume,src=pgdata,dst=/var/lib/postgresql/data postgres:16
docker run --mount type=tmpfs,dst=/run/secrets,tmpfs-size=64m app
```

## Anonymous Volumes — Gotcha
- Created when a Dockerfile declares `VOLUME` and you don't supply `-v name:path`.
- Get random hex names; pile up forever (`docker volume ls -f dangling=true`).
- ⚠️ `docker run --rm` removes them with the container; otherwise they leak.

## Volume Drivers
- Default `local`; pluggable: NFS, CIFS, cloud (rclone, EBS, etc.) for shared/network storage.

## Backup Pattern
```bash
docker run --rm --volumes-from db -v $(pwd):/backup ubuntu \
  tar czf /backup/db-$(date +%F).tgz /var/lib/postgresql/data
```
- 💡 For Postgres/MySQL prefer `pg_dump`/`mysqldump` over filesystem snapshots of a running DB.

## Tags
[[Docker]] [[Volumes]] [[Storage]]
