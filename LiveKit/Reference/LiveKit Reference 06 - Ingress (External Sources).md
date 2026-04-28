# 06 — Ingress (External Sources)

🔑 Ingress imports non-WebRTC streams (RTMP, WHIP, HLS, SRT, file URLs) into a room — auto-transcoded with optional simulcast layers.

Source: https://docs.livekit.io/home/ingress/overview/

## Supported sources

- **RTMP / RTMPS** (push) — OBS, FFmpeg, GStreamer.
- **WHIP** (push) — WebRTC-HTTP Ingestion Protocol.
- **HTTP media** (pull) — HLS, MP4, MOV, MKV/WEBM, OGG, MP3, M4A.
- **SRT** (pull) — Secure Reliable Transport.

## Push vs pull

- **Push (RTMP/WHIP)**: create ingress → receive URL + stream key → user points encoder at it. Resource persists across disconnects, reusable.
- **Pull (HTTP/SRT)**: ingress fetches and publishes immediately on creation; one-shot.

## Create RTMP ingress

```python
from livekit import api
from livekit.protocol.ingress import (
    CreateIngressRequest, IngressInput, IngressVideoOptions, IngressAudioOptions,
)

req = CreateIngressRequest(
    input_type=IngressInput.RTMP_INPUT,
    name="game-stream",
    room_name="support-1234",
    participant_identity="streamer-bot",
    participant_name="Streamer",
    video=IngressVideoOptions(name="video", source=0),
    audio=IngressAudioOptions(name="audio", source=0),
)

info = await lkapi.ingress.create_ingress(req)
print(info.url, info.stream_key)
# Point OBS to: <url>  with stream key <stream_key>
```

## Create WHIP ingress

```python
req = CreateIngressRequest(
    input_type=IngressInput.WHIP_INPUT,
    name="whip-stream",
    room_name="support-1234",
    participant_identity="whip-bot",
    enable_transcoding=True,
)
info = await lkapi.ingress.create_ingress(req)
# WHIP endpoint: info.url, bearer token: info.stream_key
```

## URL pull ingress

```python
req = CreateIngressRequest(
    input_type=IngressInput.URL_INPUT,
    url="https://example.com/stream.m3u8",
    room_name="support-1234",
    participant_identity="hls-bot",
)
await lkapi.ingress.create_ingress(req)
```

## List / update / delete

```python
from livekit.protocol.ingress import (
    ListIngressRequest, UpdateIngressRequest, DeleteIngressRequest,
)

await lkapi.ingress.list_ingress(ListIngressRequest(room_name="support-1234"))
await lkapi.ingress.delete_ingress(DeleteIngressRequest(ingress_id="IN_xxx"))
```

## Encoder config notes

- For OBS: paste URL into Server, stream key into Stream Key, prefer x264, keyframe interval 2s.
- Simulcast layers configured via `IngressVideoEncodingOptions.layers`.

## Tags
[[LiveKit]] [[Reference]] [[Ingress]] [[RTMP]] [[WHIP]]
