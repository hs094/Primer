# 09 — Grafana

🔑 The single pane: query any backend (Prom, Loki, Tempo, Postgres, OTLP) and render dashboards + alerts.

Source: https://grafana.com/docs/grafana/latest/

## Datasources (built-in)
| Type | Examples |
|---|---|
| Metrics | [[Prometheus]], Mimir, InfluxDB, CloudWatch |
| Logs | [[Loki]], Elasticsearch, CloudWatch Logs |
| Traces | [[Tempo]], Jaeger, Zipkin |
| SQL | Postgres, MySQL, MSSQL, SQLite |
| OTLP | Direct OTLP receiver datasource (preview) |

## Dashboards
- Built from **panels**. Each panel: 1 datasource + 1+ queries + 1 visualization.
- Common panel types: time series, stat, gauge, bar gauge, table, heatmap, logs, traces, node graph, geomap.
- Saved as JSON → version-controllable, provisioned via files or Terraform.

## Variables (Templating)
```
$service     = label_values(up, service)
$environment = constant(prod, staging, dev)
$interval    = interval (1m, 5m, 1h auto)
```
Reference in queries: `rate(http_requests_total{service="$service"}[$__rate_interval])`.

## Alerting (unified, since Grafana 8)
- **Rule** = query + threshold + duration ("for 5m").
- **Contact points**: Slack, PagerDuty, OpsGenie, webhook, email.
- **Notification policies** route by labels (`severity=page` → PagerDuty, else Slack).
- ⚠️ Prefer Grafana-managed rules over datasource-managed when mixing backends; only one place to edit.

## Explore Mode
- Ad-hoc queries with no dashboard. Pivot: log line → trace → metric via shared `trace_id` label.

## Provisioning Tips
- 💡 Keep dashboards in git as JSON; mount via `/etc/grafana/provisioning/dashboards/`.
- 💡 Use a `prod` folder + RBAC; lock production dashboards from edits.

## Tags
[[Grafana]] [[Prometheus]] [[Loki]] [[Tempo]] [[Dashboards]]
