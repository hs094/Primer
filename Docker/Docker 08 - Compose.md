# 08 — Compose
🔑 Compose is one YAML file describing a multi-container app — `docker compose up` brings up the stack with a project network, named volumes, and dependency ordering.

Source: https://docs.docker.com/compose/

## Minimal `docker-compose.yml`
```yaml
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: dev
    volumes: [pgdata:/var/lib/postgresql/data]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      retries: 10
  api:
    build: .
    env_file: .env
    depends_on:
      db: { condition: service_healthy }
    ports: ["8000:8000"]
volumes:
  pgdata:
```

## Top-Level Keys
| Key | Purpose |
|---|---|
| `services` | Containers (image or build) |
| `networks` | Custom networks (a default project net is auto-created) |
| `volumes` | Named volumes |
| `configs` / `secrets` | Mounted config / secret files |
| `profiles` | Tag services so they only start with `--profile` |

## depends_on + Healthchecks
- Bare `depends_on: [db]` only waits for **start**, not readiness.
- Use `condition: service_healthy` with a real `healthcheck` to wait until the service answers.

## Override Files
- `docker-compose.yml` (base) + `docker-compose.override.yml` (auto-merged in dev).
- `docker compose -f base.yml -f prod.yml up` to layer prod overrides explicitly.
- 💡 Keep secrets and host bind mounts in the override — base stays portable.

## Tags
[[Docker]] [[Compose]] [[Healthchecks]]
