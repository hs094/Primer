# 08 — Data Messages

🔑 Six data APIs split between reliable message delivery (text/byte streams, RPC, state) and lossy continuous flow (data tracks, packets).

Source: https://docs.livekit.io/home/client/data/

## API Matrix
| API | Use Case | Delivery |
|---|---|---|
| Text Streams | Chat, LLM token streaming | Reliable, chunked, topic-routed |
| Byte Streams | Files, images, blobs | Reliable, progress events |
| RPC | Call participant method, await response | Reliable, request/response |
| Data Tracks | Sensor/telemetry feed | Lossy, realtime-prioritized |
| Data Packets | Low-level send | Per-packet `reliable` / `lossy` |
| State Sync | `attributes` + room `metadata` | Reliable, replicated |

## Data Packets (raw)
```typescript
const encoder = new TextEncoder();
await room.localParticipant.publishData(
  encoder.encode(JSON.stringify({ type: 'cursor', x: 12, y: 7 })),
  { reliable: false, topic: 'cursor', destinationIdentities: ['user-2'] },
);

room.on(RoomEvent.DataReceived, (payload, participant, kind, topic) => {
  const msg = JSON.parse(new TextDecoder().decode(payload));
});
```
`kind`: `DataPacket_Kind.RELIABLE` or `LOSSY`.

## Text Streams
```typescript
const writer = await room.localParticipant.streamText({ topic: 'chat' });
await writer.write('hello ');
await writer.write('world');
await writer.close();

room.registerTextStreamHandler('chat', async (reader, info) => {
  for await (const chunk of reader) console.log(info.identity, chunk);
});
```

## Byte Streams
```typescript
const writer = await room.localParticipant.streamBytes({
  topic: 'files',
  name: 'photo.png',
  mimeType: 'image/png',
  totalSize: bytes.length,
});
await writer.write(bytes);
await writer.close();
```

## RPC
```typescript
room.registerRpcMethod('summarize', async ({ payload, callerIdentity }) => {
  return JSON.stringify({ result: await summarize(payload) });
});

const reply = await room.localParticipant.performRpc({
  destinationIdentity: 'agent-1',
  method: 'summarize',
  payload: text,
  responseTimeout: 10_000,
});
```

## Python publishData
```python
await room.local_participant.publish_data(
    payload=b"ping",
    reliable=True,
    topic="health",
    destination_identities=["agent-1"],
)

@room.on("data_received")
def on_data(packet: rtc.DataPacket):
    print(f"from {packet.participant.identity}: {packet.data!r}")
```

## Tags
[[LiveKit]] [[DataChannel]] [[RPC]] [[Streams]]
