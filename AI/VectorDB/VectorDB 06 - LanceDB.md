# 06 — LanceDB

🔑 [[LanceDB]] is an **embedded** vector DB — no server, no daemon. Tables are directories of Lance files (columnar, versioned). Open it like SQLite, scale it like Parquet.

Source: https://lancedb.github.io/lancedb/

## What Makes It Different
- **Lance format** — columnar, zero-copy, faster random access than Parquet
- **Embedded** — `pip install lancedb`, point at a path, done
- **S3-native** — same code targets local disk, S3, GCS, Azure
- **Versioned tables** — time-travel queries (`table.checkout(version)`)
- Python, Rust, JS, Java clients on the same on-disk format

## Async Python
```python
import lancedb
from lancedb.pydantic import LanceModel, Vector

class Doc(LanceModel):
    text: str
    vector: Vector(1536)

db = await lancedb.connect_async("s3://my-bucket/lancedb")
table = await db.create_table("docs", schema=Doc)
await table.add([{"text": "...", "vector": vec}])
hits = await (
    table.search(query_vec).where("text LIKE '%llm%'").limit(10).to_pandas()
)
```

## Indexing
HNSW + IVF-PQ both available. ⚠️ For < ~100k rows you don't need an index at all — Lance's columnar scan is fast enough that exact kNN beats the index-build cost.

```python
await table.create_index(metric="cosine", index_type="IVF_PQ",
                         num_partitions=256, num_sub_vectors=96)
```

## When To Reach For It
| Use case | Fit |
|---|---|
| Local RAG prototype | Excellent — zero infra |
| Notebook + S3 archive | Excellent — cheap cold storage |
| Multi-tenant SaaS, 1000 QPS | Wrong tool — use Qdrant/Weaviate |
| Co-located with PyTorch training data | Excellent — Lance is a torch DataLoader source too |

💡 Mental model: LanceDB is to vector DBs what DuckDB is to OLAP — embedded, file-based, surprisingly far up the scale curve.

## Tags
[[VectorDB]] [[LanceDB]] [[ANN]] [[RAG]]
