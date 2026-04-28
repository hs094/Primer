# 05 — Egress (Recording and Streaming)

🔑 Egress exports room/track media to MP4, HLS, or RTMP livestreams (YouTube/Twitch/Facebook) via Chrome+GStreamer pipelines.

Source: https://docs.livekit.io/home/egress/overview/

## Egress types

- **RoomComposite**: full-room mix rendered through a Chrome web template — best for meeting recordings.
- **Web**: records any URL, independent of any LiveKit room.
- **Participant**: single participant's audio + video, mixed.
- **TrackComposite**: one audio + one video track synchronized.
- **Track**: raw single-track export, no transcoding.
- **Auto egress**: triggered automatically on room creation via room config.

## Outputs

- File output: S3, GCS, Azure, AliOSS, or local.
- Stream output: RTMP/RTMPS to YouTube Live, Twitch, Facebook Live.
- Segments: HLS playlist + MP4 segments.

## Start a room composite to S3

```python
from livekit import api
from livekit.protocol.egress import (
    RoomCompositeEgressRequest, EncodedFileOutput, EncodedFileType,
)
from livekit.protocol.egress import S3Upload

req = RoomCompositeEgressRequest(
    room_name="support-1234",
    layout="speaker",
    audio_only=False,
    file_outputs=[
        EncodedFileOutput(
            file_type=EncodedFileType.MP4,
            filepath="recordings/{room_name}-{time}.mp4",
            s3=S3Upload(
                access_key="AKIA...",
                secret="...",
                bucket="my-recordings",
                region="us-east-1",
            ),
        )
    ],
)

info = await lkapi.egress.start_room_composite_egress(req)
print(info.egress_id)
```

## Start RTMP livestream

```python
from livekit.protocol.egress import StreamOutput, StreamProtocol

req = RoomCompositeEgressRequest(
    room_name="support-1234",
    stream_outputs=[
        StreamOutput(
            protocol=StreamProtocol.RTMP,
            urls=["rtmp://a.rtmp.youtube.com/live2/STREAM-KEY"],
        )
    ],
)
await lkapi.egress.start_room_composite_egress(req)
```

## Stop / list

```python
from livekit.protocol.egress import StopEgressRequest, ListEgressRequest

await lkapi.egress.stop_egress(StopEgressRequest(egress_id="EG_xxx"))
resp = await lkapi.egress.list_egress(ListEgressRequest(room_name="support-1234"))
```

## Architecture

Composite requests spin up a headless Chrome rendering a web template; track requests use the SDK directly. GStreamer encodes and pushes to file/stream sinks. Self-host requires the egress service deployed alongside LiveKit.

## Tags
[[LiveKit]] [[Reference]] [[Egress]] [[Recording]] [[Streaming]]
