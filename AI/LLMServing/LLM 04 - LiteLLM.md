# 04 — LiteLLM

🔑 LiteLLM normalizes 100+ providers (OpenAI, Anthropic, Bedrock, Vertex, Azure, [[vLLM]], [[Ollama]]) behind the OpenAI Chat Completions schema. Use the SDK for code; run the **proxy server** as a centralized [[Gateway]].

Source: https://docs.litellm.ai/

## Two Modes
| Mode | When |
|---|---|
| **Python SDK** (`litellm.completion`) | One service, few providers, want library-level control |
| **Proxy server** (`litellm --config`) | Many teams/services share keys, need cost limits, virtual keys, audit logs |

## SDK Usage
```python
from litellm import acompletion

resp = await acompletion(
    model="anthropic/claude-sonnet-4-5",   # or "openai/gpt-4o", "bedrock/...", "ollama/llama3.1"
    messages=[{"role": "user", "content": "ping"}],
    fallbacks=["openai/gpt-4o-mini"],       # auto-retry on failure
)
```
Same call shape regardless of backend. Streaming, tool calls, vision all supported uniformly.

## Proxy Config (`config.yaml`)
```yaml
model_list:
  - model_name: smart
    litellm_params:
      model: anthropic/claude-sonnet-4-5
      api_key: os.environ/ANTHROPIC_API_KEY
  - model_name: cheap
    litellm_params:
      model: openai/gpt-4o-mini
router_settings:
  routing_strategy: latency-based-routing
  fallbacks: [{"smart": ["cheap"]}]
general_settings:
  master_key: sk-master-...
  database_url: os.environ/DATABASE_URL
```
Run `litellm --config config.yaml` → OpenAI-compatible endpoint at `:4000`.

## What the Proxy Adds
- 🔑 **Virtual keys** per team/user with budgets and rate limits.
- **Cost tracking** persisted to Postgres; per-key spend dashboards.
- **Routing** — latency-based, weighted, simple-shuffle across deployments.
- **Fallbacks** — model A fails → try model B with same request shape.
- ⚠️ **Caching** — Redis prompt-response cache, careful with stale answers.

💡 Centralizing LLM calls behind LiteLLM lets you swap models per-tenant without code changes. See [[Gateway Patterns]].

## Tags
[[LiteLLM]] [[Gateway]] [[Routing]] [[Fallbacks]] [[Cost Tracking]]
