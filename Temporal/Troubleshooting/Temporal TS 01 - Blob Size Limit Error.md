# TS 01 — Blob Size Limit Error

🔑 **`BlobSizeLimitError` — your payload (Workflow/Activity input or result) crossed Temporal's 2 MB single-request or 4 MB transaction limit.**

## The error

```
BlobSizeLimitError: blob size exceeds limit
```

Common variants in logs:

```
WorkflowExecutionTerminated: BlobSizeLimitError: ResponseSize ... exceeds limit
ActivityTaskFailed: serialized result exceeds blob size limit
```

If a **Workflow Task response** crosses the 4 MB ceiling, Temporal terminates the Workflow with a non-recoverable `WorkflowExecutionTerminated` event — retries will not help.

## What the limits actually are

| Scope | Limit |
|---|---|
| Single request payload (one input/result) | 2 MB |
| Workflow Task transaction (sum of new events in one task) | 4 MB |

Why they exist: the docs state the limits "prevent excessive resource usage and potential performance issues when handling large payloads" against [[Temporal EN 07 - Event History|Event History]] and the persistence layer.

## Root causes (ranked by what we actually see)

1. **Fan-out in one Workflow Task** — scheduling thousands of activities or child workflows in a single iteration; the response event batch alone busts 4 MB.
2. **Big single payload** — passing a whole document, image, or query result as an Activity argument or return.
3. **Memo / Search Attribute bloat** — stuffing large strings into [[Temporal DV 21 - Data Handling|memos]].
4. **Accumulated state** — appending to a list inside Workflow state until the next Activity input crosses 2 MB.

## Fix 1 — batch the work

Stop scheduling everything at once.

```python
# Bad: 10k activities in one Workflow Task → response > 4 MB
results = await asyncio.gather(*[
    workflow.execute_activity(process, item) for item in items
])

# Good: chunk and await
BATCH = 100
results = []
for chunk in chunked(items, BATCH):
    batch = await asyncio.gather(*[
        workflow.execute_activity(process, x) for x in chunk
    ])
    results.extend(batch)
    await asyncio.sleep(0.001)  # gives the Workflow Task room to close
```

Workflow-level batching: split into child workflows and `await` each.
Workflow-Task-level: small batches with a tiny sleep between them so each task closes under the limit.

## Fix 2 — compress with a codec

A [[Temporal PD 03 - Codecs and Encryption|Payload Codec]] can transparently gzip every payload. Useful when payloads are ~1.5–2 MB but compress well (JSON, logs).

⚠️ Compression is a delaying tactic, not a fix. If raw data keeps growing, you'll cross 2 MB compressed eventually.

## Fix 3 — external storage with reference passing

Move the bytes out of Temporal entirely.

```python
@activity.defn
async def transcode(input_ref: S3Ref) -> S3Ref:
    raw = await s3.get(input_ref)
    out = transcode_bytes(raw)
    return await s3.put(out)  # returns small {bucket, key} dict
```

Workflow inputs/outputs become references (a few hundred bytes); the actual blob lives in S3/GCS/Blob storage. The codec pattern from [[Temporal PD 03 - Codecs and Encryption|PD 03]] can automate this.

⚠️ Trade-off: every encode/decode now depends on object storage health.

## Fix 4 — rethink Workflow shape

If a Workflow legitimately needs to hold MBs of state, you're using Temporal as a database. Consider:

- `continue_as_new` to reset Event History before payloads accumulate.
- A child workflow per logical chunk, with the parent only holding aggregate metadata.
- Moving derived results to an external store keyed by Workflow ID.

## Diagnosing it in prod

```promql
# how often blob limits are tripped
sum(rate(temporal_request_failure_total{status_code="ResourceExhausted"}[5m])) by (operation)
```

Web UI: open the Workflow, look at the failing event — it'll be a `WorkflowTaskFailed` or `WorkflowExecutionTerminated` with `BlobSizeLimit` in the cause.

## Footguns

⚠️ Termination from a 4 MB Workflow Task is **non-recoverable**. Retries / signals cannot save it; you have to re-run the Workflow with smaller batching after fixing the code.
⚠️ "It works in staging" — staging payloads are usually smaller. Test with prod-shaped data.
⚠️ Aggregating Activity results inside the Workflow without ever flushing — the next Activity input is the sum so far.
⚠️ Compressing solves *one* doubling of capacity, not unbounded growth.

## Quick decision tree

```
big payload?
├── many small items in one task → BATCH
├── individually large item       → EXTERNAL STORAGE (codec ref-pass)
├── compresses well               → CODEC compression (interim)
└── unbounded growth in state     → continue_as_new / redesign
```

💡 **Takeaway:** Batch fan-out, push large blobs to object storage, and use a codec for compression — never rely on raw payloads near the 2 MB / 4 MB ceiling.
