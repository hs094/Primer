# 13 — SLOs & Error Budgets

🔑 Pick a user-visible SLI, set an SLO target, alert on burn rate (not raw errors).

Source: https://sre.google/workbook/alerting-on-slos/

## SLI / SLO / SLA
| | Definition | Example |
|---|---|---|
| **SLI** | Quantitative measure of service quality | `successful_requests / total_requests` |
| **SLO** | Internal target on the SLI | "99.9% over 30 days" |
| **SLA** | External contract; SLO + consequences (refunds) | "99.5% or 10% credit" |

## Error Budget
- `error_budget = 1 − SLO`. At 99.9% SLO → 0.1% budget.
- 30-day budget at 1M requests/day = 30,000 allowed failures.
- **Spend it.** 100% reliability is wasteful — ship faster while inside budget; freeze deploys when burning.

## Burn Rate
```
burn_rate = (errors_in_window / requests_in_window) / (1 − SLO)
```
A burn rate of **1** = exactly exhaust budget by end of period. **14.4** = exhaust 30-day budget in 2 days.

## Multi-Window, Multi-Burn-Rate Alerts
| Severity | Long window | Short window | Burn rate | Budget consumed |
|---|---|---|---|---|
| Page  | 1h  | 5m  | 14.4 | 2%  |
| Page  | 6h  | 30m | 6    | 5%  |
| Ticket | 3d  | 6h  | 1    | 10% |

Alert iff **both** windows exceed threshold → fewer false pages, faster recovery.

## PromQL Sketch
```promql
# 1h burn rate for 99.9% SLO (= 0.001 budget)
(
  sum(rate(http_requests_total{status=~"5.."}[1h]))
  / sum(rate(http_requests_total[1h]))
) / 0.001 > 14.4
```

## Practical Rules
- 💡 SLI must reflect **user pain** — measure at the LB or client, not deep internals.
- 💡 Use percentiles for latency (`p99 < 500ms`), not averages.
- ⚠️ Low-traffic services break burn-rate math; aggregate or use synthetic probes.
- ⚠️ Don't gold-plate SLOs to 99.999% unless contracts require it — exhausts ops budget on diminishing returns.

## Reference
- *Site Reliability Engineering* book, ch. 4 — https://sre.google/sre-book/service-level-objectives/

## Tags
[[SLO]] [[SLI]] [[Error Budget]] [[Prometheus]] [[Grafana]] [[Alerting]]
