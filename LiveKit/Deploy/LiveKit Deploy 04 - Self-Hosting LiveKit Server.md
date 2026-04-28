# 04 — Self-Hosting LiveKit Server (Local)

🔑 `livekit-server --dev` boots a single-node SFU on `127.0.0.1:7880` with hardcoded `devkey`/`secret` credentials — zero-config local dev, never for prod.

Source: https://docs.livekit.io/home/self-hosting/local/

## Install

```bash
# macOS
brew update && brew install livekit

# Linux
curl -sSL https://get.livekit.io | bash

# Windows: download from https://github.com/livekit/livekit/releases
```

## Run Dev Mode

```bash
livekit-server --dev
```

Binds `127.0.0.1:7880`. Defaults:

| Setting | Value |
|---------|-------|
| API key | `devkey` |
| API secret | `secret` |
| WebSocket URL | `ws://localhost:7880` |
| ICE/UDP range | `50000-60000` |

## Expose to LAN

```bash
livekit-server --dev --bind 0.0.0.0
```

Then mobile devices on the same network can reach `ws://<host-lan-ip>:7880`.

## Config File

For anything beyond `--dev`, use a YAML config:

```yaml
# livekit.yaml
port: 7880
bind_addresses:
  - ""

rtc:
  tcp_port: 7881
  port_range_start: 50000
  port_range_end: 60000
  use_external_ip: true   # required on cloud VMs

keys:
  APIxxxxxxxxxxxx: secretxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

# optional: distributed mode
redis:
  address: localhost:6379

# optional: prometheus
prometheus:
  port: 6789
```

Run with:

```bash
livekit-server --config livekit.yaml
```

Generate a key/secret pair:

```bash
livekit-server generate-keys
```

## Docker

```bash
docker run --rm \
  -p 7880:7880 -p 7881:7881 -p 50000-60000:50000-60000/udp \
  -v $PWD/livekit.yaml:/livekit.yaml \
  livekit/livekit-server --config /livekit.yaml
```

Use host networking on Linux for best WebRTC behavior:

```bash
docker run --rm --network host \
  -v $PWD/livekit.yaml:/livekit.yaml \
  livekit/livekit-server --config /livekit.yaml
```

## Ports Cheat Sheet

| Port | Proto | Purpose |
|------|-------|---------|
| 7880 | TCP | Signaling (WebSocket) + HTTP |
| 7881 | TCP | ICE/TCP fallback |
| 50000-60000 | UDP | Media (RTP/RTCP) |
| 5349 | TCP | TURN/TLS (prod) |
| 3478 | UDP | TURN/UDP (prod) |
| 6789 | TCP | Prometheus metrics |

## Tags
[[LiveKit]] [[Deploy]] [[Self-Hosting]]
