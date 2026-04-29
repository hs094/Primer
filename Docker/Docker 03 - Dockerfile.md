# 03 — Dockerfile
🔑 Declarative recipe: each instruction produces a layer; pick the right verb (COPY vs ADD, ENTRYPOINT vs CMD) and order for cache.

Source: https://docs.docker.com/engine/reference/builder/

## Instruction Cheat Sheet
| Instruction | Use |
|---|---|
| `FROM` | Base image; first non-comment line; multiple = multi-stage |
| `RUN` | Execute at build time; creates a layer |
| `COPY` | Copy files from build context — prefer this |
| `ADD` | Like COPY + auto-extracts tar + supports URLs (rarely needed) |
| `WORKDIR` | `cd` for subsequent steps; creates dir |
| `ENV` | Persistent env var in image and runtime |
| `ARG` | Build-time only variable |
| `EXPOSE` | Documentation; doesn't publish ports |
| `HEALTHCHECK` | Periodic command → `healthy`/`unhealthy` |
| `USER` | Drop to non-root for following steps + runtime |
| `ENTRYPOINT` | Fixed executable; `docker run` args become its args |
| `CMD` | Default args (overridden by `docker run … cmd`) |

## ENTRYPOINT vs CMD
- `ENTRYPOINT ["python","app.py"]` + `CMD ["--debug"]` → runs `python app.py --debug`; user can override only `CMD`.
- 💡 Use exec form (JSON array) — shell form spawns `/bin/sh -c` and breaks signals.

## .dockerignore
- Excludes paths from build context sent to the daemon.
- Speeds builds + avoids leaking secrets/`node_modules`/`.git` into images.
```
.git
node_modules
**/__pycache__
.env*
```

## Tags
[[Docker]] [[Dockerfile]] [[BuildKit]]
