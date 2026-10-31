# Reliability and observability

## Telemetry (OpenTelemetry everywhere)
- Traces: auto-instrument HTTP, DB, Kafka; propagate W3C `traceparent` through APIs and events.
- Logs: structured JSON with timestamp, level, service, tenant_id, trace_id, span_id, request_id, event. No PII/secrets.
- Metrics: RED per endpoint and consumer (Rate, Errors, Duration histogram) + USE for resources.
- Business metrics per story (e.g. `invoices_created_total`, `collections_amount_inr`, `reminders_sent_total`).
- Metric names snake_case with unit suffix; labels low-cardinality (never user id or invoice id).

## LLM observability (ai-agents service — required from MVP0)

Every LLM call through the ai-agents service must emit the following metrics and spans. Add these to the service template so they are present from the first story.

**Metrics (Mimir via OTel):**
- `llm_requests_total{service, model, intent, tenant_tier, status}` — counter
- `llm_tokens_input_total{service, model, intent}` — counter (prompt tokens)
- `llm_tokens_output_total{service, model, intent}` — counter (completion tokens)
- `llm_latency_seconds{service, model, intent}` — histogram (p50/p95/p99)
- `llm_cost_inr{service, model, tenant_id}` — counter (tokens × model price × ₹/$ rate, updated daily)
- `llm_eval_pass_rate{eval_set, model}` — gauge (from CI eval harness)
- `llm_guardrail_blocks_total{reason}` — counter

**Spans (Tempo):**
- One span per LLM call: `llm.chat` with attributes `llm.model`, `llm.input_tokens`, `llm.output_tokens`, `llm.latency_ms`, `llm.intent`, `tenant_id`. No PII in span attributes — masking runs before the call.
- Tool calls as child spans: `llm.tool_call` with `tool.name`, `tool.success`.

**Alerts:**
- LLM error rate > 2% for 5 min → P2.
- LLM p95 latency > 5 s for 10 min → P3.
- Per-tenant daily cost exceeds budget → P2 + soft cap (429 QUOTA_EXCEEDED).
- Eval pass rate drops below 80% in CI → failing gate, blocks deploy.

**Dashboard:** cost/day, tokens/day, p95 latency, error rate, top intents, per-tenant cost breakdown, eval trend over time.

**Why:** At scale, LLM cost is the largest variable cost driver. Without per-tenant metering, overruns are invisible until the bill arrives. Per-tenant data also powers the AI credits usage meter in billing.

## SLOs (defaults; story may tighten)
| Signal | Target |
|---|---|
| API availability | 99.9% / 30 days |
| Read p95 / p99 | 300 ms / 800 ms |
| Write p95 / p99 | 500 ms / 1.5 s |
| Event end-to-end lag p95 | 5 s |
| Kill switch propagation | ≤ 10 s |
| LLM p95 latency (first token) | < 3 s |
| LLM eval pass rate | ≥ 85% on 200-prompt set |

## Alerts
- Alert on SLO burn rate (multi-window: 2%/1 h fast, 5%/6 h slow), not raw CPU.
- Also: DLQ > 0, consumer lag > threshold, breaker open > 1 min, replica lag > 30 s.
- **Audit consumer lag: alert threshold ≤ 10 s** (tighter than the standard 60 s — audit falling behind during an incident must be known before the postmortem, not after).
- Each alert links a runbook in `docs/runbooks/` and routes to on-call; pages create an incident.

## Dashboards
- Grafana dashboard per service (RED, SLO, dependencies, Kafka lag) as code in the repo.
- Each story adds/updates its panel.
- LLM dashboard: cost, tokens, latency, eval pass rate (see LLM observability above).

## Resilience
- Timeouts on every network call (default connect 1 s, total 3 s; tune per dependency).
- Retries only for idempotent ops: max 3, exponential backoff + jitter; respect Retry-After.
- Circuit breakers on every external and inter-service call, with a defined fallback (cache, degraded mode, queue for later, clear error).
- Bulkheads: separate pools per dependency. Rate limiting and load shedding at the gateway.
- Health: `/healthz` (liveness) and `/readyz` (readiness incl. dependencies).
- Graceful shutdown: stop intake, drain in-flight, commit offsets.
