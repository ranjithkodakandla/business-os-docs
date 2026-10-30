# Reliability and observability

## Telemetry (OpenTelemetry everywhere)
- [ ] Traces: auto-instrument HTTP, DB, Kafka; propagate W3C `traceparent` through APIs **and** events.
- [ ] Logs: structured JSON with `timestamp, level, service, tenant_id, trace_id, span_id, request_id, event`. No PII/secrets.
- [ ] Metrics: **RED** per endpoint and consumer (Rate, Errors, Duration histogram) + USE for resources.
- [ ] Business metrics per story (e.g. `invoices_created_total`, `collections_amount_inr`, `reminders_sent_total`).
- [ ] Metric names `snake_case` with unit suffix; labels low-cardinality (never user id or invoice id).

## SLOs (defaults; story may tighten)
| Signal | Target |
|---|---|
| API availability | 99.9% / 30 days |
| Read p95 / p99 | 300 ms / 800 ms |
| Write p95 / p99 | 500 ms / 1.5 s |
| Event end-to-end lag p95 | 5 s |
| Kill switch propagation | ≤ 10 s |

## Alerts
- [ ] Alert on SLO burn rate (multi-window: 2%/1 h fast, 5%/6 h slow), not raw CPU.
- [ ] Also: DLQ > 0, consumer lag > threshold, breaker open > 1 min, replica lag > 30 s.
- [ ] Each alert links a runbook in `docs/runbooks/` and routes to on-call; pages create an incident.

## Dashboards
- [ ] Grafana dashboard per service (RED, SLO, dependencies, Kafka lag) as code in the repo.
- [ ] Each story adds/updates its panel.

## Resilience
- [ ] **Timeouts** on every network call (default connect 1 s, total 3 s; tune per dependency).
- [ ] **Retries** only for idempotent ops: max 3, exponential backoff + jitter; respect `Retry-After`.
- [ ] **Circuit breakers** on every external and inter-service call, with a defined fallback (cache, degraded mode, queue for later, clear error).
- [ ] Bulkheads: separate pools per dependency. Rate limiting and load shedding at the gateway.
- [ ] Health: `/healthz` (liveness) and `/readyz` (readiness incl. dependencies).
- [ ] Graceful shutdown: stop intake, drain in-flight, commit offsets.
