# 08 — Object Store

🔑 Object Store is [[JetStream]] for blobs — chunks large files into stream messages, exposes an S3-ish API.

Source: https://docs.nats.io/nats-concepts/jetstream/obj_store

## Model
- A **bucket** is backed by a stream `OBJ_<bucket>` (data) + `OBJ_<bucket>_meta` (metadata).
- Each object is split into fixed-size chunks (default 128 KiB) published as ordered messages.
- Metadata: name, size, sha256 digest, content-type, custom headers.
- **Watch** the bucket for `put`/`delete` events.

## Python
```python
obs = await js.create_object_store(bucket="artifacts")

# Upload
with open("model.bin", "rb") as f:
    info = await obs.put("model-v2.bin", f)
print(info.size, info.digest)

# Download
async for chunk in await obs.get("model-v2.bin"):
    sink.write(chunk)

# List + delete
async for entry in await obs.list():
    print(entry.name, entry.size)
await obs.delete("model-v1.bin")
```

## vs S3
| | NATS Object Store | S3 |
|---|---|---|
| Scale | bounded by stream storage on the cluster | effectively unlimited |
| Latency | sub-ms inside cluster | tens of ms |
| Multi-region | leaf nodes / mirrors | native |
| Cost model | self-hosted | per-GB-month + egress |
| Eventing | watch (push) | EventBridge / SQS notifications |

⚠️ **Not a distributed object store.** All chunks of a bucket must fit on each replica's disk. Fine for ML models, configs, build artifacts — wrong tool for petabytes of media.

💡 Killer use case: ship config bundles + binaries to [[Edge]] leaf nodes alongside live messaging on the same connection.

## Tags
[[NATS]] [[JetStream]] [[Edge]]
