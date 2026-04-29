# 10 — Gateway Patterns

🔑 An **LLM gateway** sits between your apps and provider APIs to centralize keys, budgets, routing, caching, and observability. Without one, every service reimplements retries, fallbacks, and cost tracking — badly.

## What a Gateway Does
| Concern | Why centralize |
|---|---|
| **Virtual keys** | One real provider key, many scoped tokens (per team, per tenant) |
| **Budgets / rate limits** | Hard caps per key/user/model — stop runaway spend |
| **Routing** | Latency-based, weighted, or per-tenant model selection |
| **Fallbacks** | Provider down → reroute without app code change |
| **Prompt caching** | Redis-backed exact-match cache cuts repeat costs |
| **Observability** | Every request logged with cost, latency, tokens, user |
| **Key rotation** | Rotate provider keys without redeploying apps |

## Options
| Tool | Shape | Best For |
|---|---|---|
| **[[LiteLLM]] proxy** | Self-hosted, OpenAI-compatible, Postgres-backed | Most teams — open source, full feature set |
| **Portkey** | SaaS or self-host, similar feature set | Teams wanting hosted dashboards |
| **Helicone** | Observability-first proxy | Teams primarily wanting logs/analytics |
| **OpenRouter** | Hosted aggregator with credits | Solo devs, no infra |
| **Cloudflare AI Gateway** | Edge proxy, caching | Already on Cloudflare |

💡 Self-host LiteLLM unless you have a strong reason — your data and keys stay yours, and the SDK and proxy share the same interface.

## Minimal LiteLLM Gateway Setup
```yaml
# config.yaml
model_list:
  - model_name: chat-default
    litellm_params: {model: anthropic/claude-sonnet-4-5, api_key: os.environ/ANTHROPIC_API_KEY}
  - model_name: chat-cheap
    litellm_params: {model: openai/gpt-4o-mini, api_key: os.environ/OPENAI_API_KEY}
router_settings:
  fallbacks: [{"chat-default": ["chat-cheap"]}]
general_settings:
  master_key: os.environ/LITELLM_MASTER_KEY
  database_url: os.environ/DATABASE_URL
litellm_settings:
  cache: true
  cache_params: {type: redis, host: os.environ/REDIS_HOST}
```
Apps point at `http://gateway:4000/v1` with virtual keys minted via `/key/generate`.

## Anti-patterns
- ⚠️ Each microservice holding raw provider keys → key sprawl, no spend visibility.
- ⚠️ App-level retries on 429 without jitter → thundering herd when provider blips.
- ⚠️ Caching responses on **non-deterministic** prompts (high temperature) → stale, surprising answers.

🧪 Track cost per **business action** (e.g. per chat message, per generation), not per request. The gateway makes this trivial.

## Tags
[[Gateway]] [[LiteLLM]] [[Observability]] [[Cost Tracking]] [[Routing]]
