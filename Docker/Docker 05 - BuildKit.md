# 05 — BuildKit
🔑 BuildKit is the modern build backend — concurrent stage graph, content-addressed cache, and `--mount` types for caches and secrets.

Source: https://docs.docker.com/build/buildkit/

## What Changed vs Legacy Builder
- **Concurrent solver** — independent stages build in parallel; unused stages skipped.
- **Content-addressed cache** — checksum-based, transferable across hosts/CI.
- **Incremental context** — only changed files re-uploaded.
- Default in modern Docker; invoked via `docker buildx build` (or just `docker build`).

## Cache Mounts
Persist compiler/package caches between builds without baking them into layers.
```dockerfile
# syntax=docker/dockerfile:1.7
FROM python:3.13-slim
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt
```
- 💡 Speeds up `pip`, `npm`, `apt`, `go mod`, `cargo` rebuilds dramatically.

## Secret Mounts
Inject secrets at build time without leaving them in any layer.
```bash
docker build --secret id=npmrc,src=$HOME/.npmrc .
```
```dockerfile
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc \
    npm ci
```
- ⚠️ Never `COPY` secrets — they live in the image forever, even after `RUN rm`.

## buildx
- Frontend that drives BuildKit; manages **builder instances** (local, remote, k8s).
- Multi-platform: `docker buildx build --platform linux/amd64,linux/arm64 -t img:tag --push .`.
- 🧪 `docker buildx bake` runs grouped builds from `docker-bake.hcl`.

## Tags
[[Docker]] [[BuildKit]] [[buildx]]
