# audit — MVP0/MVP1 design (DRAFT)

> **DRAFT — not frozen.** Design only: no OpenAPI, nothing provisioned. Open decisions OD1–OD9 are **UNCONFIRMED**. Mismatches with the Architect Doc are flagged, not resolved.

Sources: [Sheet tab 'MVP0-1 Design'](https://docs.google.com/spreadsheets/d/18p-aJ0QeW6L3SJ-CNSrNCjwKWBpVdkQAfkzeKOg40Oc/edit#gid=119904232) · [BOS Architect's Doc](https://docs.google.com/document/d/1sKoivvz3JLkKPI1Jta0i6XWnWuW57-rSwWICmm1w9VM/edit) · Jira SCRUM (MVP0 + MVP1 epics SCRUM-70..89, stories SCRUM-90..143). Generated 31 Oct 2026 IST.

Conventions, error/idempotency rules, quality gates, open decisions, mismatches and gaps: see [README](README.md). Rows: MVP0 4, MVP1 8.

## MVP0

### MVP0 — Data model (DRAFT)

| Row ID | Item | Detail | Epic ID | Story ID | Jira key | Constraints | Mismatch / note |
|---|---|---|---|---|---|---|---|
| TBL-AU-01 | audit_events (append-only, monthly partitions) | tenant_id, id, at, actor, actor_type(USER\|AGENT), action, entity, diff jsonb; 1-year retention; UPDATE/DELETE revoked | EP-MVP0-05 | ST-MVP0-05-03 | [SCRUM-35](https://rozvaa.atlassian.net/browse/SCRUM-35) | C1, C13 |  |
| TBL-AU-02 | platform_audit_events (GLOBAL) | platform admin actions (kill switch, impersonation) | EP-MVP0-05 | ST-MVP0-05-03 | [SCRUM-35](https://rozvaa.atlassian.net/browse/SCRUM-35) | C1 | Cross-tenant → allowlist. |

### MVP0 — API contracts (DRAFT)

| Row ID | Item | Detail | Role / permission | Idempotency-Key | Emits / consumes | Epic ID | Story ID | Jira key | Constraints | Mismatch / note | p95 latency target | Payload limit |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| API-AU-01 | GET /v1/audit-events?entity=&from=&to=&cursor= | → {items[{id, at, actor, action, entity, diff}], next_cursor}. RANGE_TOO_WIDE 422 | OWNER/ADMIN/ACCOUNTANT | n/a | consumes all domain events | EP-MVP0-05 | ST-MVP0-05-03 | [SCRUM-35](https://rozvaa.atlassian.net/browse/SCRUM-35) | C13 | Arch tab says /v1/audit; Doc matches story. | <300 ms @50 rps (story DoD) | 1 MB (proposed) |

### MVP0 — Observability (DRAFT)

| Row ID | Item | Detail | Epic ID | Story ID | Jira key | Constraints | Mismatch / note |
|---|---|---|---|---|---|---|---|
| OBS-AUDIT | RED + business metrics | RED per endpoint; business: events_written, write_lag_s; JSON logs PII-masked (TST-06) with trace_id, hashed tenant_id; OTel spans incl. DB/Kafka/outbound; SLO: no loss (outbox lag <60 s) (proposed unless story); alerts: burn-rate 2%/1h & 5%/6h, DLQ>0, breaker open; Grafana panel "audit" | EP-MVP0-06 | ST-MVP0-06-01, ST-MVP0-06-02 | [SCRUM-22](https://rozvaa.atlassian.net/browse/SCRUM-22), [SCRUM-28](https://rozvaa.atlassian.net/browse/SCRUM-28) | C12 | proposed |

## MVP1

### MVP1 — Data model (DRAFT)

| Row ID | Item | Detail | Epic ID | Story ID | Jira key | Constraints |
|---|---|---|---|---|---|---|
| M1-TBL-AUDI-01 | (none — telemetry/tracing are platform config) | tenant_id + RLS policy; PostHog event plan only, PII-free | EP-MVP1-13 | ST-MVP1-13-01 | [SCRUM-104](https://rozvaa.atlassian.net/browse/SCRUM-104) (epic [SCRUM-82](https://rozvaa.atlassian.net/browse/SCRUM-82)) | C1, C3, C15 |

### MVP1 — Feature flags (with OFF behaviour) (DRAFT)

| Row ID | Item | Detail | Epic ID | Story ID | Jira key | Constraints | Mismatch / note | Flag OFF behaviour |
|---|---|---|---|---|---|---|---|---|
| M1-FLG-13-01 | plat.obs.13_01 | PostHog product telemetry (PII-free tracking plan) | EP-MVP1-13 | ST-MVP1-13-01 | [SCRUM-104](https://rozvaa.atlassian.net/browse/SCRUM-104) (epic [SCRUM-82](https://rozvaa.atlassian.net/browse/SCRUM-82)) | C9 | key format <module>.<area>.<NN_NN> differs from MVP0 proposed keys — M1-MM-09 | OFF: hidden / 404 FEATURE_DISABLED; data readable, no writes lost (story) |
| M1-FLG-13-02 | plat.obs.13_02 | Distributed tracing, uptime/status page and on-call alerting | EP-MVP1-13 | ST-MVP1-13-02 | [SCRUM-105](https://rozvaa.atlassian.net/browse/SCRUM-105) (epic [SCRUM-82](https://rozvaa.atlassian.net/browse/SCRUM-82)) | C9 | key format <module>.<area>.<NN_NN> differs from MVP0 proposed keys — M1-MM-09 | OFF: hidden / 404 FEATURE_DISABLED; data readable, no writes lost (story) |

### MVP1 — Resilience (DRAFT)

| Row ID | Item | Detail | Epic ID | Story ID | Jira key | Constraints | Mismatch / note |
|---|---|---|---|---|---|---|---|
| M1-RES-13-01 | cb.posthog_api | timeout 10 s external; breaker 50%/20 calls, half-open 30 s; bulkhead 10/tenant; fallback: degrade to manual / template text, never block the core flow | EP-MVP1-13 | ST-MVP1-13-01 | [SCRUM-104](https://rozvaa.atlassian.net/browse/SCRUM-104) (epic [SCRUM-82](https://rozvaa.atlassian.net/browse/SCRUM-82)) | C6, C12 | story |
| M1-RES-13-02 | cb.grafana_tempo | timeout 10 s external; breaker 50%/20 calls, half-open 30 s; bulkhead 10/tenant; fallback: degrade to manual / template text, never block the core flow | EP-MVP1-13 | ST-MVP1-13-02 | [SCRUM-105](https://rozvaa.atlassian.net/browse/SCRUM-105) (epic [SCRUM-82](https://rozvaa.atlassian.net/browse/SCRUM-82)) | C6, C12 | story |

### MVP1 — Observability (DRAFT)

| Row ID | Item | Detail | Epic ID | Story ID | Jira key | Constraints | Mismatch / note |
|---|---|---|---|---|---|---|---|
| M1-OBS-AUDIT | business metrics: none stated | RED + logs + spans + SLO + alerts + Grafana panel per story (values above) | EP-MVP1-13 | ST-MVP1-13-01, ST-MVP1-13-02 | [SCRUM-104](https://rozvaa.atlassian.net/browse/SCRUM-104), [SCRUM-105](https://rozvaa.atlassian.net/browse/SCRUM-105) | C12 | story |

### MVP1 — Infra (DRAFT)

| Row ID | Item | Detail | Epic ID | Story ID | Jira key | Constraints | Mismatch / note |
|---|---|---|---|---|---|---|---|
| M1-INF-13-01 | PostHog telemetry (infra/SDK) | Server + web SDK; region/self-host = OPEN QUESTION. | EP-MVP1-13 | ST-MVP1-13-01 | [SCRUM-104](https://rozvaa.atlassian.net/browse/SCRUM-104) (epic [SCRUM-82](https://rozvaa.atlassian.net/browse/SCRUM-82)) | C8, C12 | no API contract; IaC via reviewed PR only |
| M1-INF-13-02 | Tracing, status page, on-call | OTel collector; 10% + 100% errors sampling (ASSUMPTION). | EP-MVP1-13 | ST-MVP1-13-02 | [SCRUM-105](https://rozvaa.atlassian.net/browse/SCRUM-105) (epic [SCRUM-82](https://rozvaa.atlassian.net/browse/SCRUM-82)) | C8, C12 | no API contract; IaC via reviewed PR only |
