# 08 — Chroma

🔑 Chroma's pitch is the smallest possible API surface. Two modes — embedded or client-server — same Python interface either way.

Source: https://docs.trychroma.com/

## Modes
| Client | Storage | Use |
|---|---|---|
| `chromadb.Client()` | In-memory, lost on exit | Notebooks, tests |
| `chromadb.PersistentClient(path=...)` | Local SQLite + parquet | Single-process apps |
| `chromadb.HttpClient(host, port)` | Remote Chroma server | Multi-process / shared |
| `chromadb.AsyncHttpClient(...)` | Same, async | FastAPI services |

## Minimal Python
```python
import chromadb

client = chromadb.PersistentClient(path="./chroma")
coll = client.get_or_create_collection(name="docs")
coll.add(
    ids=["d1", "d2"],
    documents=["pineapple grows on bushes", "oranges are citrus"],
    metadatas=[{"src": "wiki"}, {"src": "wiki"}],
)
res = coll.query(query_texts=["tropical fruit"], n_results=2,
                 where={"src": "wiki"})
```

## Async Server Mode
```python
import chromadb
client = await chromadb.AsyncHttpClient(host="localhost", port=8000)
coll = await client.get_or_create_collection("docs")
await coll.add(ids=[...], embeddings=[...], documents=[...])
res = await coll.query(query_embeddings=[query_vec], n_results=10)
```

⚠️ Chroma will embed text **client-side** by default using a small ONNX model. Pass `embeddings=[...]` directly when you've already vectorized — otherwise you'll silently ship every doc through that fallback model.

## Strengths / Weaknesses
| Pro | Con |
|---|---|
| Trivial API, fast to learn | Fewer index knobs than Qdrant/Weaviate |
| Embedded → server with one line change | Single-node server, no native sharding |
| Apache 2.0, Chroma Cloud as managed option | Smaller scale ceiling |

💡 Pick Chroma for prototypes and small/medium apps; graduate to [[Qdrant]] or [[LanceDB]] when index size or QPS bites.

## Tags
[[VectorDB]] [[ANN]] [[RAG]]
