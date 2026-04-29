# 05 — Semantic Conventions

🔑 Standardized attribute names so dashboards/queries work identically across services, languages, vendors.

Source: https://opentelemetry.io/docs/specs/semconv/

## Why It Matters
- Without it: `http.method` vs `httpMethod` vs `request_method` — every dashboard rebuilt per service.
- With it: one Grafana panel `rate(http_server_request_duration_seconds_count{http.response.status_code=~"5.."}[5m])` works everywhere.

## Common Attribute Namespaces
| Namespace | Examples |
|---|---|
| `http.*` | `http.request.method`, `http.response.status_code`, `http.route` |
| `db.*` | `db.system` (`postgresql`/`redis`), `db.statement`, `db.operation` |
| `messaging.*` | `messaging.system` (`kafka`/`rabbitmq`), `messaging.destination.name` |
| `rpc.*` | `rpc.system`, `rpc.service`, `rpc.method` |
| `server.*` / `client.*` | `server.address`, `client.port` |
| `exception.*` | `exception.type`, `exception.message`, `exception.stacktrace` |
| `gen_ai.*` | `gen_ai.system`, `gen_ai.request.model`, `gen_ai.usage.*_tokens` |

## Resource Attributes (per-process, not per-span)
- `service.name` (required), `service.version`, `service.namespace`
- `deployment.environment.name` (`prod`/`staging`)
- `host.*`, `k8s.pod.name`, `cloud.region`

## Stability Tiers
- **Stable** — won't break (HTTP, DB, RPC).
- **Experimental** — may rename (Gen-AI evolved fast in 2024–25).

## Pragmatic Rules
- 💡 Don't invent `myco.user.id` if `enduser.id` exists.
- ⚠️ Avoid high-cardinality attrs as **metric** dimensions (user_id, request_id) — fine on spans, lethal on Prometheus.
- 🧪 `pip install opentelemetry-semantic-conventions` exposes constants: `from opentelemetry.semconv.attributes import http_attributes`.

## Tags
[[OpenTelemetry]] [[Semantic Conventions]] [[Prometheus]] [[Grafana]]
