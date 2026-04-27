# PD 03 — Codecs and Encryption

🔑 **A Payload Codec encrypts/compresses bytes between your Workers and the [[Temporal EN 11 - Temporal Service|Temporal Service]] so payloads exist plaintext only on hosts you control.**

## Why TLS is not enough

TLS protects data **in transit**. It does nothing for data **at rest in [[Temporal EN 07 - Event History|Event History]]** or visible in the Web UI. If a Workflow input contains a credit card and you only have TLS, the Service's persistence layer stores it plaintext.

A **Payload Codec** closes that gap: payloads are encrypted on the client/Worker before they cross the wire, and the Service stores the ciphertext.

## Pipeline

```
your value
   │  Data Converter (default JSON serialization)
   ▼
Payload (bytes + metadata)
   │  Payload Codec (your compress/encrypt)
   ▼
Payload' (encoded bytes + new metadata.encoding)
   │  gRPC + TLS
   ▼
Temporal Service (stores Payload' as-is)
```

## What the codec covers

The codec applies to every payload that flows through Temporal:

- Workflow / Activity / Child Workflow inputs and results
- Signal inputs
- [[Temporal DV 21 - Data Handling|Memo]] fields
- Query inputs and results
- Local Activity and Side Effect results

⚠️ It does **not** cover Search Attributes — those must remain plaintext (the Service indexes them).

## Implementing a custom codec

Recommendation from the docs: encrypt the *entire input Payload* and wrap it in a new Payload whose `metadata.encoding` identifies your codec.

Sketch (Python):

```python
class EncryptionCodec(PayloadCodec):
    async def encode(self, payloads: list[Payload]) -> list[Payload]:
        return [
            Payload(
                metadata={"encoding": b"binary/encrypted", "key-id": self.kid},
                data=aes_gcm_encrypt(p.SerializeToString(), self.key),
            )
            for p in payloads
        ]

    async def decode(self, payloads: list[Payload]) -> list[Payload]:
        out = []
        for p in payloads:
            if p.metadata.get("encoding") != b"binary/encrypted":
                out.append(p); continue
            inner = aes_gcm_decrypt(p.data, self.key_for(p.metadata["key-id"]))
            decoded = Payload(); decoded.ParseFromString(inner)
            out.append(decoded)
        return out
```

Plug it into the Client's Data Converter:

```python
client = await Client.connect(
    target_host,
    namespace="prod",
    data_converter=dataclasses.replace(
        pydantic_data_converter,
        payload_codec=EncryptionCodec(key=settings.codec_key),
    ),
)
```

⚠️ Both the SDK Client *and* every Worker must use the same codec (and have the keys). A Worker without the key cannot decode inputs.

## Codec Server (decoding for the Web UI / CLI)

The Web UI and `temporal` CLI fetch raw payloads from the Service — i.e. ciphertext. To make them human-readable for operators, run a **Codec Server**: an HTTP service that exposes your codec's `decode` (and optionally `encode`) endpoints.

Required endpoints:

- `POST /decode` — required.
- `POST /encode` — optional (only if you want round-trip from UI).

Required headers on requests:

- `Content-Type: application/json`
- `X-Namespace: <namespace>`
- `Authorization: ...` (optional, recommended)

CORS for browser-initiated UI calls:

```
Access-Control-Allow-Origin: https://cloud.temporal.io
Access-Control-Allow-Methods: POST
Access-Control-Allow-Headers: Content-Type, X-Namespace, Authorization
```

### Wiring the UI / CLI

Web UI: namespace admin sets the codec endpoint per Namespace; can opt to forward the user's JWT.

CLI:

```bash
# global
temporal env set --codec-endpoint https://codec.internal/decode

# per command
temporal workflow show --workflow-id w1 --codec-endpoint https://codec.internal/decode
```

## Securing the Codec Server

⚠️ The Codec Server holds (or can fetch) decryption keys. Treat it like a secrets vault.

- Keep it on private VPN / VPC; never expose plain HTTP to the internet.
- Terminate HTTPS, require auth (mTLS, OAuth/Auth0, JWT verification).
- Authorize per-Namespace via the `X-Namespace` header.
- Log access; rotate keys; restrict who can hit `/encode`.

## Key management

- Keys live in **your** KMS / secrets manager — Vault, AWS KMS, GCP KMS, etc.
- Tag each Payload with a `key-id` so you can rotate without breaking history.
- Old keys must remain decryptable for the lifetime of any open Workflow plus the Namespace retention window.

## Beyond encryption: large-payload offload

A codec can also stash payloads in object storage and replace them with a small reference, sidestepping Temporal's 2 MB / 4 MB limits ([[Temporal TS 01 - Blob Size Limit Error|TS 01]]):

```
encode:  Payload(data=large_blob)   →  upload to S3, return Payload(data=s3://bucket/key)
decode:  Payload(data=s3://...)     →  fetch and rehydrate
```

⚠️ Offload increases blast radius — every encode/decode now depends on S3 health. Apply only to Workflows where payloads truly exceed the limit.

## Operational footguns

⚠️ Forgetting to ship the codec to **every** Worker — old Workers crash on decode.
⚠️ No codec server → Web UI shows opaque ciphertext, on-call cannot debug.
⚠️ Embedding key material in container images instead of fetching from KMS at start-up.
⚠️ Encrypting Search Attributes — they must stay plaintext for indexing.
⚠️ Rotating keys without preserving the old key for in-flight Workflows.

💡 **Takeaway:** TLS protects the wire; a Payload Codec protects the storage. Implement codec + run a hardened Codec Server before any sensitive data hits a production Workflow.
