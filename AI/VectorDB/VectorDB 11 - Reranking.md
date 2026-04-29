# 11 — Reranking

🔑 Retrieval gets you 100 plausible candidates. A **reranker** is a heavier model that re-scores those 100 against the query and picks the top 5–10 you actually feed the LLM.

## Bi-Encoder vs Cross-Encoder
| | Bi-encoder (retrieval) | Cross-encoder (rerank) |
|---|---|---|
| Encodes | Query and doc **separately** | Query and doc **together** |
| Output | Two vectors → cosine | Single relevance score |
| Speed | Embed once, ANN over millions | Forward pass per (q, d) pair |
| Quality | Good | Materially better |

⚠️ A cross-encoder cannot be precomputed — it must run at query time. That's why you only feed it the top-K from a fast retriever, never the whole corpus.

## The Cascade
```
query
  ↓ embed (bi-encoder)
ANN search → top 100 candidates
  ↓ optional: hybrid fusion (RRF)
cross-encoder rerank → top 10
  ↓
LLM context
```

## Cohere Rerank
```python
import cohere
co = cohere.AsyncClientV2()
res = await co.rerank(
    model="rerank-v4.0-pro",
    query="how do diffusion models train",
    documents=[hit.payload["text"] for hit in candidates],
    top_n=10,
)
ranked = [candidates[r.index] for r in res.results]
```

## Self-Hosted: BGE Reranker
```python
from FlagEmbedding import FlagReranker
reranker = FlagReranker("BAAI/bge-reranker-v2-m3", use_fp16=True)
scores = reranker.compute_score([(query, doc) for doc in docs])
```

`bge-reranker-v2-m3` is multilingual, ~568M params; `-large` (~560M) and `-base` (~278M) are the speed/quality knobs.

## Latency Budget
ANN top-100: 5–20 ms. Cross-encoder × 100 on GPU: 50–150 ms. Cohere API rerank: 100–300 ms.

💡 Common pattern: retrieve top-100 hybrid → rerank to top-20 → send top-5–10 to the LLM. Rerank is the highest-leverage [[RAG]] quality fix once embeddings are decent.

## Tags
[[VectorDB]] [[Embeddings]] [[RAG]] [[ANN]]
