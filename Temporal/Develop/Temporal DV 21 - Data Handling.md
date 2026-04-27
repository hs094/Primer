# DV 21 — Data Handling

🔑 Every byte of input, output, signal, query, and exception that crosses the Worker ↔ Service boundary is processed by the **Data Converter**. Customize it to support non-JSON types (Pydantic, protobuf), to encrypt payloads end-to-end, or to offload huge blobs to external storage.

## The three layers

```
Application object
      │
      ▼
┌────────────────┐
│ PayloadConverter │  required — object → bytes (JSON by default)
└────────────────┘
      │
      ▼
┌────────────────┐
│ PayloadCodec    │  optional — encryption / compression
└────────────────┘
      │
      ▼
┌────────────────┐
│ ExternalStorage │  optional — offload large payloads
└────────────────┘
      │
      ▼
Bytes on the wire / in History
```

Only the **PayloadConverter** is required; the other two are opt-in.

| Layer | Purpose | Default |
|---|---|---|
| **PayloadConverter** | Serialize app data to bytes | JSON |
| **PayloadCodec** | Transform encoded payloads | none (passthrough) |
| **ExternalStorage** | Offload large payloads | none (stored in Workflow History) |

⚠️ The Service **never sees** decrypted/decoded payloads — the codec runs Worker-side. This means tools like the Web UI can't render encrypted payloads without a separate decoder hooked up.

## When to customize

- **Non-JSON types** — Pydantic models, protobuf messages, dataclasses with custom fields → custom **PayloadConverter**.
- **Encryption / compression** → custom **PayloadCodec**.
- **Large payloads** (multi-MB, e.g. ML inputs) that would bloat Workflow History → **ExternalStorage**.

## Wiring a custom Data Converter

Pass `data_converter=` to the Client; the Worker inherits it through the Client.

```python
from temporalio.client import Client
from temporalio.converter import DataConverter

client = await Client.connect(
    "localhost:7233",
    data_converter=my_data_converter,
)
```

The same `data_converter=` argument applies to test environments and Workers built around the Client.

## Pydantic example (PayloadConverter)

To use Pydantic models as Workflow inputs/outputs, pair the `pydantic_data_converter` with a sandbox passthrough so Pydantic isn't reloaded on every Workflow run:

```python
from temporalio.client import Client
from temporalio.contrib.pydantic import pydantic_data_converter
from temporalio.worker import Worker
from temporalio.worker.workflow_sandbox import (
    SandboxedWorkflowRunner, SandboxRestrictions,
)

client = await Client.connect(
    "localhost:7233",
    data_converter=pydantic_data_converter,
)

worker = Worker(
    client,
    task_queue="q",
    workflows=[MyWorkflow],
    activities=[my_activity],
    workflow_runner=SandboxedWorkflowRunner(
        restrictions=SandboxRestrictions.default.with_passthrough_modules("pydantic"),
    ),
)
```

See [[Temporal DV 20 - Sandbox and Sync vs Async]] for why passthrough matters here.

## Encryption (PayloadCodec)

A `PayloadCodec` exposes `encode(payloads)` and `decode(payloads)` — both run **Worker-side**, so keys never leave your infrastructure.

Skeleton:

```python
from temporalio.api.common.v1 import Payload
from temporalio.converter import PayloadCodec

class EncryptionCodec(PayloadCodec):
    def __init__(self, key: bytes):
        self.key = key

    async def encode(self, payloads: list[Payload]) -> list[Payload]:
        # encrypt each payload's data, return new payloads with metadata flag
        ...

    async def decode(self, payloads: list[Payload]) -> list[Payload]:
        # reverse the operation
        ...
```

Then wrap it in a Data Converter:

```python
from temporalio.converter import DataConverter, default

data_converter = DataConverter(
    payload_converter_class=default().payload_converter_class,
    payload_codec=EncryptionCodec(key=KEY),
)
client = await Client.connect("localhost:7233", data_converter=data_converter)
```

⚠️ Codec runs on **every** payload — keep it fast. A slow codec adds latency to every Activity input/output, every Signal, every Query.

## Codec server (Web UI decryption)

Because the Service stores ciphertext, the Web UI shows opaque blobs. Run a **codec server** — a tiny HTTP service that decodes payloads on demand — and point the UI at it. Your team sees plaintext in the UI; the Service still only stores ciphertext.

## External storage

For payloads that would balloon Workflow History (think 10MB ML model inputs), implement an `ExternalStorage` interface that:

1. On encode: writes the payload to S3/GCS/etc., replaces the payload bytes with a small reference.
2. On decode: fetches by reference, returns the real bytes.

This keeps History compact while preserving the illusion of in-line payloads to Workflow code.

## Serialization rules of thumb

- **Keep payloads small.** Workflow History is replayed; large payloads slow replay and bloat storage.
- **Stable shapes.** Adding fields is fine; renaming/removing breaks replay of old histories. Use new field names with defaults.
- **Don't ship secrets without a codec.** Unencrypted secrets in payloads end up in history, exports, and audit logs.
- **Use [[Temporal DV 21 - Data Handling]] custom converters in tests too** — the test Client must use the same converter.

## Cross-references

- [[Temporal DV 16 - Temporal Client]] — where the converter is configured
- [[Temporal DV 20 - Sandbox and Sync vs Async]] — passthrough for Pydantic et al.
- [[Temporal DV 18 - Observability]] — logs/metrics for codec performance
- [[Temporal EN 02 - Temporal SDKs]] — what the SDK ships out of the box

💡 **Takeaway:** The Data Converter has three layers — converter (required, JSON default), codec (optional, encryption), external storage (optional, large payloads). Use `pydantic_data_converter` for Pydantic models with sandbox passthrough; codecs run Worker-side so keys stay local; pair encryption with a codec server for Web UI decryption.
