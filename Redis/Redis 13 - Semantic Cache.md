# 13 — Semantic Cache

🔑 Cache LLM responses by embedding similarity, not exact-match keys. A new prompt within the threshold of a prior prompt returns the prior answer — sub-ms latency, $0 token cost.

Source: https://redis.io/docs/latest/develop/ai/

## Why Semantic, Not Exact
- Exact-match cache (`md5(prompt) → response`) misses on every wording variation.
- Embed the prompt → vector lookup → if cosine distance ≤ threshold, return the cached answer.
- Big wins: chatbots, RAG, agentic loops with repeated tool calls.

## RedisVL `SemanticCache`
```python
from redisvl.extensions.llmcache import SemanticCache
from redisvl.utils.vectorize import OpenAITextVectorizer

cache = SemanticCache(
    name="llmcache",
    redis_url="redis://localhost:6379",
    distance_threshold=0.1,           # 0.0 = identical, 1.0 = orthogonal (cosine)
    ttl=3600,
    vectorizer=OpenAITextVectorizer(model="text-embedding-3-small"),
)

# read-through pattern
hit = await cache.acheck(prompt=user_q, num_results=1)
if hit:
    return hit[0]["response"]

answer = await llm.complete(user_q)
await cache.astore(prompt=user_q, response=answer, metadata={"user": uid})
return answer
```

## Threshold Tuning
| Distance | Behaviour |
|---|---|
| `0.05` | Near-paraphrase only — high precision, low hit rate |
| `0.10` | Reasonable default for chat / FAQ |
| `0.20` | Loose — risk wrong-answer reuse |

- 💡 Eval on a held-out set: log queries + cache hits, manually grade for "would I have served this?"
- ⚠️ Threshold lives in the *embedding space* of your vectorizer — change models = re-tune.

## TTL Strategy
- Short TTL (minutes) for time-sensitive answers ("what's the weather").
- Long TTL (hours-days) for stable knowledge ("explain CRC16").
- Per-entry TTL via `ttl=` on `astore`. Pair with `metadata` to invalidate by tenant / user.

## Variants
| Tool | Idea |
|---|---|
| `SemanticCache` | Cache full prompt → response |
| `SemanticRouter` (RedisVL) | Route prompt → intent / tool by nearest centroid |
| `MessageHistory` | Store chat turns with vector + recency retrieval |

## ⚠️ Gotchas
- Cache poisoning: if you store a hallucinated answer, every paraphrase gets it. Filter / grade before `astore`.
- PII: prompts often contain user data. Either don't cache them, or scope by `metadata={"user": uid}` and filter on retrieve.
- Embedding cost: every miss does an embed call. Pick a cheap embedding model.

## Tags
[[Redis]] [[Semantic Cache]] [[RedisVL]] [[Vector Search]] [[LLM]]
