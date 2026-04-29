# 05 — Weaviate

🔑 [[Weaviate]] is **schema-first**: you declare collections + vectorizer modules, and Weaviate does the embedding call for you on insert and query.

Source: https://docs.weaviate.io/weaviate

## Modules — "DB-Side Embeddings"
A vectorizer module turns an insert into `embed(text) → store`. Pick one per collection.

| Module | Provider |
|---|---|
| `text2vec-openai` | OpenAI embeddings |
| `text2vec-cohere` | Cohere |
| `text2vec-weaviate` | Weaviate-managed |
| `text2vec-transformers` | Self-hosted HF |
| `multi2vec-clip` | CLIP, image+text |

⚠️ DB-side embedding is convenient but couples your DB tier to a third-party API and its rate limits. Many teams disable modules and pre-embed in app code.

## Python v4 Client
```python
import weaviate
from weaviate.classes.config import Configure, Property, DataType

with weaviate.connect_to_weaviate_cloud(cluster_url=URL, auth_credentials=KEY) as client:
    client.collections.create(
        name="Article",
        vector_config=Configure.Vectors.text2vec_openai(),
        properties=[
            Property(name="title", data_type=DataType.TEXT),
            Property(name="body",  data_type=DataType.TEXT),
        ],
    )
    coll = client.collections.use("Article")
    coll.data.insert({"title": "...", "body": "..."})
    res = coll.query.near_text(query="diffusion models", limit=5)
```

## Multi-Tenancy
Built-in. Each tenant is a hot/cold-able shard — far cheaper than per-tenant collections.
```python
coll.tenants.create([Tenant(name="acme"), Tenant(name="globex")])
coll.with_tenant("acme").data.insert(...)
```

## GraphQL
The HTTP API is GraphQL-first; the v4 Python client wraps it. Useful for exploring in the console:
```graphql
{ Get { Article(nearText: {concepts: ["llm scaling"]}, limit: 3) { title body } } }
```

💡 Pick Weaviate when you want batteries-included (vectorizers, generative modules, multi-tenancy) and don't mind a heavier server.

## Tags
[[VectorDB]] [[Weaviate]] [[Embeddings]] [[RAG]]
