# EN 14 — Extensibility

🔑 **Four extension points let you customize how data flows through the SDK: Data Conversion, Context Propagation, Interceptors, and Plugins.**

## The Four Mechanisms

Temporal SDKs are intentionally small at the core. Cross-cutting concerns — encryption, tracing, multi-tenant headers, custom serialization — plug in through four orthogonal extension points:

1. **Data Conversion** — customize how arguments and return values are serialized, compressed, or encrypted.
2. **Context Propagation** — pass custom metadata (tracing IDs, tenant IDs, auth tokens) across Workflow, Activity, and Child Workflow boundaries.
3. **Interceptors** — add cross-cutting behavior (observability, authorization, header manipulation) before and after SDK operations.
4. **Plugins** — bundle interceptors, context propagators, data converters, and built-in definitions into a single reusable package.

## Data Conversion

A **Data Converter** is the SDK component that handles serialization and encoding of data entering and exiting a Temporal Service. It encodes application values into **Payloads** before transmission, and decodes payloads back into language values on receipt.

### Payload

A Payload is binary data plus metadata describing how the bytes should be interpreted (e.g. content type, encoding, encryption key id). Every workflow input, activity argument, signal value, and query response is a Payload on the wire.

### Payload Converter

Customize this to change **how values become bytes**. Useful for:
- Switching JSON libraries / Protobuf / MessagePack.
- Handling SDK-specific types not covered by the default.

### Payload Codec

Customize this to apply **transformations on the bytes** after conversion:
- **Encryption** — encrypt payloads so the Temporal Service stores ciphertext only.
- **Compression** — shrink large payloads.
- Multiple codecs can be chained.

⚠️ Sensitive data should be encrypted via a Codec so it's plaintext only on your trusted hosts. The Temporal Service stores whatever bytes the codec emits — if you don't encrypt, operators reading event histories see plaintext.

### Failure Converter

Maps language-native exceptions to/from Temporal's wire failure type, so failures cross workflow / activity / child / signal boundaries with structured information preserved.

## Context Propagation

Context Propagators carry metadata across SDK boundaries that workflows themselves can't otherwise observe — the propagation runs at the SDK layer.

Typical uses:
- Distributed tracing IDs (OpenTelemetry).
- Tenant IDs in multi-tenant systems.
- Auth tokens / request IDs for downstream calls.

The propagator extracts context from the caller, encodes it into headers attached to the workflow/activity task, and re-injects it on the receiving side. This works across:
- Client → Workflow start.
- Workflow → Activity invocation.
- Workflow → Child Workflow start.
- Workflow → Signal / Update / Query.

⚠️ Workflows are deterministic. A propagator that injects values that change across replays (timestamps, random IDs generated locally) will break replay. Pull such values via Activities, not through the propagator.

## Interceptors

Interceptors wrap SDK operations to add behavior **before** and **after** them — the SDK's middleware model.

Two axes:

- **Worker interceptors** — wrap workflow / activity execution inside a worker.
- **Client interceptors** — wrap calls a Client makes (start workflow, signal, query, update).

Within each, interceptors can be **inbound** (intercepting calls coming in) or **outbound** (intercepting calls going out). A single chain often covers all four directions.

Use cases:
- Emit metrics / traces / logs around every workflow and activity invocation.
- Inject and validate auth headers.
- Strip or redact sensitive arguments from logs.
- Tag every outbound call with tenant context.

⚠️ Interceptors run on **every** invocation. Heavy synchronous work (network calls, large allocations) in an interceptor degrades worker throughput. Keep them tight and offload work to activities when possible.

## Plugins

A **Plugin** packages a coherent set of the above — interceptors, context propagators, data converters, and any built-in workflow / activity definitions — into one installable unit.

Plugins are how third-party integrations ship: an OpenTelemetry plugin, a Vault encryption plugin, a multi-tenant plugin. Apps install the plugin once, and all four extension axes are wired correctly.

## Choosing the Right Extension

| Need | Mechanism |
|---|---|
| Change how values become bytes | Payload Converter |
| Encrypt / compress bytes on the wire | Payload Codec |
| Map exceptions across boundaries | Failure Converter |
| Carry tenant ID / trace ID end-to-end | Context Propagator |
| Wrap every SDK call with metrics/auth | Interceptor |
| Ship a reusable bundle of all of the above | Plugin |

## Cross-references

- What gets serialized: [[Temporal EN 03 - Workflows]], [[Temporal EN 04 - Activities]]
- Data on the wire crosses these boundaries: [[Temporal EN 08 - Workflow Message Passing]], [[Temporal EN 09 - Child Workflows]]
- Cross-namespace calls also use these extensions: [[Temporal EN 13 - Temporal Nexus]]
- Operator-level security context: [[Temporal SH 07 - Security]]

💡 **Takeaway:** Pick the smallest hook that solves the problem — Codec for encryption, Propagator for metadata, Interceptor for cross-cutting logic, Plugin to bundle them.
