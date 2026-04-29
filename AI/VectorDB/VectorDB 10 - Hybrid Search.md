# 10 — Hybrid Search

🔑 Dense vectors miss exact tokens (proper nouns, SKUs, error codes). Sparse keyword search misses paraphrase. **Hybrid** = run both, fuse the rankings.

## Why Both
| Query | Dense wins | Sparse wins |
|---|---|---|
| "how do transformers work" | ✅ | — |
| "ERR_42 retry exhausted" | — | ✅ |
| "user @hs094 last login" | — | ✅ |
| "thoughts on agentic systems" | ✅ | — |

⚠️ Pure dense looks great on academic benchmarks and worse than you'd expect on real product search where 30% of queries are codes, IDs, or rare terms.

## Sparse Side
- **BM25** — classic IDF-weighted term match (Lucene/Tantivy under the hood)
- **SPLADE / uniCOIL** — neural sparse: learned term weights from a transformer; integrates as a sparse vector

## Fusion: Reciprocal Rank Fusion
RRF ignores raw scores (which aren't comparable across systems), uses ranks:
```
rrf_score(d) = Σ_i 1 / (k + rank_i(d))   # k typically 60
```
Order documents by `rrf_score` desc. That's the whole algorithm.

## Qdrant Example
```python
await client.query_points("docs",
    prefetch=[
        models.Prefetch(query=models.SparseVector(indices=idx, values=val),
                        using="sparse", limit=50),
        models.Prefetch(query=dense_vec, using="dense", limit=50),
    ],
    query=models.FusionQuery(fusion=models.Fusion.RRF),
    limit=10,
)
```

## Weaviate Example
```python
res = coll.query.hybrid(
    query="ERR_42 retry exhausted",
    alpha=0.5,          # 0 = pure BM25, 1 = pure vector
    limit=10,
)
```

💡 `alpha` is fine for prototyping; RRF with prefetched candidates is sturdier in production because each side keeps its own tuning. Pair hybrid retrieve with [[VectorDB 11 - Reranking|reranking]] for the cleanest top-K.

## Tags
[[VectorDB]] [[Qdrant]] [[Weaviate]] [[RAG]] [[ANN]]
