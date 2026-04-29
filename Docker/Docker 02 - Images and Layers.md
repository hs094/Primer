# 02 — Images and Layers
🔑 An image is an ordered stack of read-only layers merged by overlayfs; rebuilds reuse unchanged layers via content-addressed cache, so layer order = build speed.

Source: https://docs.docker.com/build/cache/

## Layered Filesystem
- Each `RUN`/`COPY`/`ADD` produces one layer (a tarball diff).
- Container start: layers stacked read-only, plus a thin writable layer on top.
- **Copy-on-write** via `overlayfs` — writes to a file copy it up to the writable layer; everything else is shared.
- 💡 Many containers from one image share the same on-disk layers → tiny per-container overhead.

## Digests and Tags
- **Digest** — `sha256:…` over the manifest; immutable, exact bytes.
- **Tag** — mutable pointer (`nginx:1.27`); can be re-pushed.
- ⚠️ Pin by digest (`nginx@sha256:…`) for reproducible builds; tags lie.

## Cache Rules
- Layer cache hits when: same base layer + same instruction + same inputs.
- Any change invalidates that layer **and every layer after it**.
- Order from least → most volatile.

## Practical Ordering
```dockerfile
FROM python:3.13-slim
WORKDIR /app
COPY requirements.txt .         # rare changes
RUN pip install -r requirements.txt
COPY . .                        # frequent changes (last)
CMD ["python", "main.py"]
```
- ⚠️ `COPY . .` before deps re-installs deps on every code change.

## Tags
[[Docker]] [[Images]] [[BuildKit]] [[overlayfs]]
