# 04 — Multi-Stage Builds
🔑 Multiple `FROM` stages let you compile in a fat builder image and copy only artifacts into a minimal runtime — smaller, faster, less attack surface.

Source: https://docs.docker.com/build/building/multi-stage/

## Pattern
```dockerfile
FROM golang:1.25 AS build
WORKDIR /src
COPY . .
RUN go build -o /out/app ./cmd/app

FROM gcr.io/distroless/static-debian12
COPY --from=build /out/app /app
ENTRYPOINT ["/app"]
```
- Each `FROM` starts a new stage; `AS <name>` makes it referenceable.
- `COPY --from=build` pulls files from a prior stage (or `--from=nginx:latest` for external images).
- Final image contains only what the last stage `COPY`s in — build tools left behind.

## Base Image Choices
| Base | Size | Trade-off |
|---|---|---|
| `debian`/`ubuntu` | 70–120 MB | Full toolchain; easy debug |
| `*-slim` | 20–80 MB | Trimmed Debian; usually fine |
| `alpine` | 5–15 MB | musl libc — ⚠️ glibc-built binaries (e.g. some Python wheels) break |
| `distroless` | 2–20 MB | No shell, no package manager; runs binary only |
| `scratch` | 0 MB | Empty; static binaries (Go) only |

## Tips
- 💡 `docker build --target build` to debug a specific stage.
- 💡 BuildKit only executes stages the target depends on → cheap to keep extra test/lint stages.
- ⚠️ Distroless = no `sh`; `docker exec -it … sh` won't work — use a debug image variant.

## Tags
[[Docker]] [[Dockerfile]] [[BuildKit]] [[Distroless]]
