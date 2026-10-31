# Design — MVP0/MVP1 (owner: BOS MVP Design) — DRAFT

> **DRAFT — not frozen.** Design only: no OpenAPI, nothing provisioned. Open decisions OD1–OD9 are **UNCONFIRMED**. Mismatches with the Architect Doc are flagged, not resolved.

Sources: [Sheet tab 'MVP0-1 Design'](https://docs.google.com/spreadsheets/d/18p-aJ0QeW6L3SJ-CNSrNCjwKWBpVdkQAfkzeKOg40Oc/edit#gid=119904232) · [BOS Architect's Doc](https://docs.google.com/document/d/1sKoivvz3JLkKPI1Jta0i6XWnWuW57-rSwWICmm1w9VM/edit) · Jira SCRUM (MVP0 + MVP1 epics SCRUM-70..89, stories SCRUM-90..143). Generated 31 Oct 2026 IST.

## Services

- [identity-tenancy](identity-tenancy.md)
- [feature-flags](feature-flags.md)
- [audit](audit.md)
- [notifications](notifications.md)
- [documents](documents.md)
- [ai-agents](ai-agents.md)
- [sales-crm](sales-crm.md)
- [billing-invoicing](billing-invoicing.md)
- [collections-payments](collections-payments.md)
- [inventory](inventory.md)
- [purchase](purchase.md)

## Overview and conventions

MVP0-1 Design — rev 2 (by service). Status: DRAFT, design PAUSED for freeze; nothing frozen, no OpenAPI YAML, nothing provisioned. Sources (read-only): Epics, Stories, Data Spec, Story Map, Architecture v1 (gid=1723970403), Architect Doc rev 2. Updated 30 Oct 2026 IST.

Conventions: every table has tenant_id uuid NOT NULL + RLS ENABLE+FORCE, policy tenant_id = current_setting('app.tenant_id')::uuid; each service has schema svc_<name> + own non-owner DB user and runs SET LOCAL app.tenant_id per transaction from the gateway-signed token context (C1, C2, C3). No tenant_id in any URL, header or body (C2). Money/GST = bigint paise, *_paise suffix (C15). Cache keys t:{tenant_id}:{service}:... (C14). Events: <domain>.<event>.v<n> on Kafka, key tenant_id:aggregate_id, via per-service outbox (C6).

Row IDs: TBL- table, API- endpoint, EVT- event, RBAC-, FLG- flag, ERR- error/idempotency rule, TST- test, UI- screen→API map, MM- mismatch, GAP- coverage gap, OD- open decision. 'NO STORY' = design row with no story. Constraint numbers C1..C15 = Architecture v1 rev 2.

1. DATA MODEL (tables) — by service

2. /v1 ENDPOINTS — by service (contracts copied from Stories; no tenant_id in URL/header/body)

3. EVENTS (Kafka, per-service outbox, all v1; envelope: event_id, type, occurred_at, tenant_id (from tx context), aggregate_id, trace_id, data)

4. RBAC (checked in each service from token claims; AI uses invoking user's permissions)

5. FEATURE FLAGS (rollout; stages INTERNAL→ALPHA→BETA→%→GA, per-tenant override, kill switch, changes audited) — entitlements separate (TBL-ID-08). Keys home.chat and collections.whatsapp come from story 03-01; others proposed.

6. ERROR & IDEMPOTENCY RULES (from story contracts — apply to all services)

7. TEST STRATEGY

8. UI SCREENS → APIs (MFE per module) + infra stories

9. MISMATCHES story contract vs Architect Doc (flagged, NOT resolved — story contract used as-is)

10. COVERAGE GAPS

11. OPEN DECISIONS — all UNCONFIRMED (Ranjith to decide)

9b. MISMATCHES added for engineering standards

12. RESILIENCE per call (Ranjith standard 1: timeout / retry / circuit breaker / bulkhead / fallback) — values proposed

13. OBSERVABILITY per service (RED + business metrics, PII-masked JSON logs, OTel spans, SLI/SLO, alerts, Grafana panel)

14. CI QUALITY & SECURITY GATES (standards 2,3,4,6,7) — apply to every service

══ MVP1 DESIGN (DRAFT, not frozen) — source: Jira [SCRUM-70](https://rozvaa.atlassian.net/browse/SCRUM-70)..143 (EP-MVP1-01..20, ST-MVP1-*) read 30 Oct 2026; story contracts used as-is; mismatches flagged M1-MM-*, not resolved

M1-1. DATA MODEL (MVP1 additions; every table tenant_id NOT NULL + RLS, svc_<name> schema; money per story = decimal(18,2) INR — see M1-MM-01)

M1-2. /v1 APIs (story contracts verbatim; envelope {"error":{code,message,details,trace_id}}; Idempotency-Key on POST 24 h per story; no tenant_id in URL/header/body except where flagged)

M1-3. EVENTS (story uses PascalCase names; mapped to <domain>.<event>.v1 — M1-MM-03; bus per OD6)

M1-4. RBAC additions

M1-5. FEATURE FLAGS — one per story (story keys verbatim). OFF per story: feature hidden / API 404 FEATURE_DISABLED; data stays readable by other modules, no writes lost. Entitlements separate (M1-TBL-IDEN-08).

M1-6. RESILIENCE per external call (story values: breaker opens at 50% failures over 20 calls, half-open 30 s; bulkhead 10 concurrent per tenant; internal calls 3 s timeout; external 10 s)

M1-7. OBSERVABILITY (story: RED bos_<module>_*; JSON logs <module>.<story>.succeeded/failed, PII-masked; OTel span > db.query > outbox.publish; SLO ASSUMPTION 99.5%/28 d, p95 <500 ms (OPEN QUESTION SLO-01); alerts err>2% 5 min page, p95>1 s 10 min ticket, DLQ>0 15 min; Grafana BOS/<epic>/<story>)

M1-8. TESTS (story test plans): Vitest/RTL, Testcontainers Postgres+RLS, Playwright 360px EN/HI, Pact consumer/provider + can-i-deploy, Stryker mutation ≥70% on critical modules, chaos (Chaos Toolkit/Toxiproxy/Litmus) in staging, 1 h soak at 2x heap growth <10%, OWASP/SAST/DAST, authz allow+deny, tenant-leak per endpoint, flag ON/OFF

M1-9. UI SCREENS → APIs

M1-10. MVP1 MISMATCHES (flagged, NOT resolved)

M1-11. MVP1 COVERAGE GAPS

## Cross-cutting rows (conventions, error/idempotency, quality gates, OD1–OD9, mismatches, gaps, shared/MFE rows)

## MVP0

### MVP0 — Data model (DRAFT)

| Row ID | Item | Detail | Epic ID | Story ID | Jira key | Constraints |
|---|---|---|---|---|---|---|
| TBL-ALL-01 | outbox, inbox_dedup (one pair per service schema) | outbox: tenant_id, id, aggregate_id, type, payload, published_at; inbox_dedup: consumer, event_id (UNIQUE) | EP-MVP0-04 | ST-MVP0-04-01, ST-MVP0-04-02 | [SCRUM-20](https://rozvaa.atlassian.net/browse/SCRUM-20), [SCRUM-21](https://rozvaa.atlassian.net/browse/SCRUM-21) | C1, C6 |

### MVP0 — API contracts (DRAFT)

| Row ID | Item | Detail | Role / permission | Idempotency-Key | Emits / consumes | Epic ID | Story ID | Jira key | Constraints | Mismatch / note | p95 latency target | Payload limit |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| API-GW-01 | API gateway routing | JWT verify, catalog lookup (Valkey 60 s), signed tenant context to services, per-tenant rate limit, request_id/traceparent | — | — | — | NO STORY | NO STORY | — | C2, C3, C12, C14 | Doc routes /v1/<service>/*; story paths have no service segment (MM-01). No story for the gateway (GAP-D). | <500 ms gateway (Doc) | 1 MB (proposed) |

### MVP0 — Resilience (DRAFT)

| Row ID | Item | Detail | Epic ID | Story ID | Jira key | Constraints | Mismatch / note |
|---|---|---|---|---|---|---|---|
| RES-GW-01 | gateway → any service | timeout 3 s; no retry on POST w/o Idempotency-Key; breaker 50% errors/20 req, 30 s open; bulkhead per upstream pool; fallback: 503 + Retry-After in error envelope | EP-MVP0-01 | ST-MVP0-01-03 | [SCRUM-24](https://rozvaa.atlassian.net/browse/SCRUM-24) | C6, C12 | proposed |
| RES-ALL-01 | → feature-flags evaluate | SDK local cache; timeout 100 ms; breaker; fallback: last-known value, else flag default (OFF) | EP-MVP0-03 | ST-MVP0-03-01 | [SCRUM-25](https://rozvaa.atlassian.net/browse/SCRUM-25) | C6, C12 | proposed |
| RES-ALL-02 | → Kafka (outbox relay / consumers) | async; relay retries forever w/ backoff; consumer 5 retries → DLQ; fallback: events stay in outbox; no user impact | EP-MVP0-04 | ST-MVP0-04-01, ST-MVP0-04-02 | [SCRUM-20](https://rozvaa.atlassian.net/browse/SCRUM-20), [SCRUM-21](https://rozvaa.atlassian.net/browse/SCRUM-21) | C6, C12 | proposed |
| RES-ALL-03 | → Valkey cache | timeout 50 ms; breaker; fallback: bypass cache, read DB | NO STORY | NO STORY | — | C6, C12 | proposed |

### MVP0 — Observability (DRAFT)

| Row ID | Item | Detail | Epic ID | Story ID | Jira key | Constraints | Mismatch / note |
|---|---|---|---|---|---|---|---|
| OBS-GATEWA | RED + business metrics | RED per endpoint; business: requests, 4xx/5xx, auth_failures; JSON logs PII-masked (TST-06) with trace_id, hashed tenant_id; OTel spans incl. DB/Kafka/outbound; SLO: availability 99.9%; p95 <500 ms (Doc) (proposed unless story); alerts: burn-rate 2%/1h & 5%/6h, DLQ>0, breaker open; Grafana panel "gateway" | EP-MVP0-06 | ST-MVP0-06-01, ST-MVP0-06-02 | [SCRUM-22](https://rozvaa.atlassian.net/browse/SCRUM-22), [SCRUM-28](https://rozvaa.atlassian.net/browse/SCRUM-28) | C12 | proposed |

### MVP0 — Tests (DRAFT)

| Row ID | Item | Detail | Epic ID | Story ID | Jira key | Constraints |
|---|---|---|---|---|---|---|
| TST-01 | Tenant-leak suite | 2 tenants seeded per service; cross-tenant read/write/insert = 0 rows / rejected; request with no tenant context returns 0 rows; gate before merge | EP-MVP0-01 | ST-MVP0-01-01, ST-MVP0-01-02 | [SCRUM-19](https://rozvaa.atlassian.net/browse/SCRUM-19), [SCRUM-23](https://rozvaa.atlassian.net/browse/SCRUM-23) | C1, C11 |
| TST-02 | Contract tests | OpenAPI contract + Pact consumer tests per service; event schema compatibility in registry | EP-MVP0-01 | ST-MVP0-01-03 | [SCRUM-24](https://rozvaa.atlassian.net/browse/SCRUM-24) | C5, C11 |
| TST-03 | Outbox/consumer | event written in same tx; relay crash → redelivery; duplicate delivery no double effect; DLQ alert | EP-MVP0-04 | ST-MVP0-04-01, ST-MVP0-04-02 | [SCRUM-20](https://rozvaa.atlassian.net/browse/SCRUM-20), [SCRUM-21](https://rozvaa.atlassian.net/browse/SCRUM-21) | C6 |
| TST-04 | Migration test | expand/contract migrations run unchanged on shared, schema-per-tenant, dedicated | EP-MVP0-01 | ST-MVP0-01-01 | [SCRUM-19](https://rozvaa.atlassian.net/browse/SCRUM-19) | C10, C11 |
| TST-05 | Security | SAST + dependency scan 0 high/critical; gitleaks; TLS 1.2+/HSTS check; secrets only in Secret Manager | EP-MVP0-05 | ST-MVP0-05-01 | [SCRUM-26](https://rozvaa.atlassian.net/browse/SCRUM-26) | C8, C11 |
| TST-06 | PII masking | log/trace/LLM prompt scrubber tests (mobile, GSTIN, PAN, email) | EP-MVP0-05 | ST-MVP0-05-02 | [SCRUM-27](https://rozvaa.atlassian.net/browse/SCRUM-27) | C12 |
| TST-07 | Observability | every service emits OTel traces/logs/metrics tagged tenant_id(hashed), service, request_id; SLO alerts fire in staging | EP-MVP0-06 | ST-MVP0-06-01, ST-MVP0-06-02 | [SCRUM-22](https://rozvaa.atlassian.net/browse/SCRUM-22), [SCRUM-28](https://rozvaa.atlassian.net/browse/SCRUM-28) | C12 |
| TST-10 | Non-functional | p95 <300 ms @50 rps per API (story DoD; Doc gateway p95 <500 ms); WCAG 2.1 AA; 360 px; Lighthouse mobile ≥80 | EP-MVP0-07 | ST-MVP0-07-02 | [SCRUM-60](https://rozvaa.atlassian.net/browse/SCRUM-60) | C12 |
| TST-11 | Connection budget | pool ≤3 per instance; CI connection-budget check | NO STORY | NO STORY | — | C11 |

### MVP0 — UI → API (DRAFT)

| Row ID | Item | Detail | Epic ID | Story ID | Jira key | Constraints |
|---|---|---|---|---|---|---|
| UI-01-04 | Company & units settings screen | Calls API-ID-03..05 | EP-MVP0-01 | ST-MVP0-01-04 | [SCRUM-43](https://rozvaa.atlassian.net/browse/SCRUM-43) | C9 |
| UI-01-06 | Document series screen | Calls API-BI-01 | EP-MVP0-01 | ST-MVP0-01-06 | [SCRUM-44](https://rozvaa.atlassian.net/browse/SCRUM-44) | C9 |
| UI-01-08 | 5-minute quick setup wizard | Calls API-ID-10, API-ID-07 | EP-MVP0-01 | ST-MVP0-01-08 | [SCRUM-55](https://rozvaa.atlassian.net/browse/SCRUM-55) | C9 |
| UI-02-04 | Login, invite-accept and users screens | Calls API-ID-01, API-ID-02, API-ID-07 | EP-MVP0-02 | ST-MVP0-02-04 | [SCRUM-56](https://rozvaa.atlassian.net/browse/SCRUM-56) | C9 |
| UI-02-05 | Permissions matrix screen | Calls API-ID-06 | EP-MVP0-02 | ST-MVP0-02-05 | [SCRUM-57](https://rozvaa.atlassian.net/browse/SCRUM-57) | C9 |
| UI-02-06 | View-as role switcher | Calls API-ID-08 | EP-MVP0-02 | ST-MVP0-02-06 | [SCRUM-58](https://rozvaa.atlassian.net/browse/SCRUM-58) | C9 |
| UI-03-03 | Platform admin: rollouts & company flag grid | Calls API-FF-01, API-FF-02 | EP-MVP0-03 | ST-MVP0-03-03 | [SCRUM-45](https://rozvaa.atlassian.net/browse/SCRUM-45) | C9 |
| UI-05-04 | Activity log screen | Calls API-AU-01 | EP-MVP0-05 | ST-MVP0-05-04 | [SCRUM-59](https://rozvaa.atlassian.net/browse/SCRUM-59) | C9 |
| UI-07-01 | Shell app with routing, guards and flags | Calls API-ID-06, API-FF-01 | EP-MVP0-07 | ST-MVP0-07-01 | [SCRUM-46](https://rozvaa.atlassian.net/browse/SCRUM-46) | C9 |
| UI-07-02 | Multilingual (English + Hindi) and responsive layouts | Calls API-ID-09 | EP-MVP0-07 | ST-MVP0-07-02 | [SCRUM-60](https://rozvaa.atlassian.net/browse/SCRUM-60) | C9 |
| UI-08-02 | Approval queue, decide with reason, recently decided | Calls API-AI-04 (+ list endpoint missing, GAP-D) | EP-MVP0-08 | ST-MVP0-08-02 | [SCRUM-61](https://rozvaa.atlassian.net/browse/SCRUM-61) | C9 |
| UI-09-04 | Home chat, voice input, history, Needs-you-today and glance | Calls API-AI-01, API-AI-03 | EP-MVP0-09 | ST-MVP0-09-04 | [SCRUM-62](https://rozvaa.atlassian.net/browse/SCRUM-62) | C9 |
| UI-10-03 | Notification feed, preferences and compliance calendar screens | Calls API-NT-01, API-BI-02 | EP-MVP0-10 | ST-MVP0-10-03 | [SCRUM-63](https://rozvaa.atlassian.net/browse/SCRUM-63) | C9 |
| UI-11-03 | Inbox screen with drafts and Raise CAPA | Calls API-NT-03, API-NT-04 (+ Raise CAPA: no API, GAP-S4) | EP-MVP0-11 | ST-MVP0-11-03 | [SCRUM-64](https://rozvaa.atlassian.net/browse/SCRUM-64) | C9 |
| UI-12-04 | Suppliers, sourcing and BOM check screens | Calls API-PU-01, API-PU-02, API-IN-01 | EP-MVP0-12 | ST-MVP0-12-04 | [SCRUM-66](https://rozvaa.atlassian.net/browse/SCRUM-66) | C9 |
| UI-13-04 | Collections sprint, follow-ups and disputes screens | Calls API-CP-01..04 | EP-MVP0-13 | ST-MVP0-13-04 | [SCRUM-67](https://rozvaa.atlassian.net/browse/SCRUM-67) | C9 |
| UI-14-03 | Connected apps hub & integrations tab | Calls API-ID-11 | EP-MVP0-14 | ST-MVP0-14-03 | [SCRUM-68](https://rozvaa.atlassian.net/browse/SCRUM-68) | C9 |

### MVP0 — Infra (DRAFT)

| Row ID | Item | Detail | Epic ID | Story ID | Jira key | Constraints |
|---|---|---|---|---|---|---|
| INF-01 | Encryption, TLS/HSTS, secrets | CMEK at rest, TLS 1.2+, Secret Manager + KMS per-tenant creds | EP-MVP0-05 | ST-MVP0-05-01 | [SCRUM-26](https://rozvaa.atlassian.net/browse/SCRUM-26) | C8, C15 |
| INF-02 | PII masking | OTel processor + LLM prompt scrubber before LiteLLM | EP-MVP0-05 | ST-MVP0-05-02 | [SCRUM-27](https://rozvaa.atlassian.net/browse/SCRUM-27) | C12, C15 |
| INF-03 | OTel + Grafana LGTM | Story title says Sentry; Doc dropped Sentry (MM-16) | EP-MVP0-06 | ST-MVP0-06-01 | [SCRUM-22](https://rozvaa.atlassian.net/browse/SCRUM-22) | C12 |
| INF-04 | Metrics, SLO dashboards, alerts | RED dashboard per service; SLOs per Doc | EP-MVP0-06 | ST-MVP0-06-02 | [SCRUM-28](https://rozvaa.atlassian.net/browse/SCRUM-28) | C12 |

### MVP0 — Error / idempotency (DRAFT)

| Row ID | Item | Detail | Idempotency-Key | Epic ID | Story ID | Jira key | Constraints |
|---|---|---|---|---|---|---|---|
| ERR-01 | Error envelope | {"error":{"code","message" (i18n),"field"?,"trace_id"}}; common 400/401/403/404 (also other-tenant ids)/409/422/429/500/503. Replaces my earlier RFC 9457 proposal. |  | EP-MVP0-01, EP-MVP0-04 | ST-MVP0-01-03, ST-MVP0-04-02 | [SCRUM-24](https://rozvaa.atlassian.net/browse/SCRUM-24), [SCRUM-21](https://rozvaa.atlassian.net/browse/SCRUM-21) | C12 |
| ERR-02 | Idempotency | POST/PUT accept Idempotency-Key (UUID, 24 h); same key+body → same response; different body → 409 IDEMPOTENCY_MISMATCH. Key store per service, tenant-scoped. | Yes | EP-MVP0-01, EP-MVP0-04 | ST-MVP0-01-03, ST-MVP0-04-02 | [SCRUM-24](https://rozvaa.atlassian.net/browse/SCRUM-24), [SCRUM-21](https://rozvaa.atlassian.net/browse/SCRUM-21) | C6 |
| ERR-03 | Pagination | cursor ?cursor=&limit= (25, max 100) → {items, next_cursor}; sort created_at desc, id |  | EP-MVP0-01, EP-MVP0-04 | ST-MVP0-01-03, ST-MVP0-04-02 | [SCRUM-24](https://rozvaa.atlassian.net/browse/SCRUM-24), [SCRUM-21](https://rozvaa.atlassian.net/browse/SCRUM-21) | C5 |
| ERR-04 | Outbound side effects | executed by worker from outbox with idempotency key = event_id; consumers dedupe via inbox_dedup |  | EP-MVP0-01, EP-MVP0-04 | ST-MVP0-01-03, ST-MVP0-04-02 | [SCRUM-24](https://rozvaa.atlassian.net/browse/SCRUM-24), [SCRUM-21](https://rozvaa.atlassian.net/browse/SCRUM-21) | C6, C7 |
| ERR-05 | Tenant context | No tenant_id accepted in URL/header/body; any client-supplied tenant_id field is rejected 400 |  | EP-MVP0-01, EP-MVP0-04 | ST-MVP0-01-03, ST-MVP0-04-02 | [SCRUM-24](https://rozvaa.atlassian.net/browse/SCRUM-24), [SCRUM-21](https://rozvaa.atlassian.net/browse/SCRUM-21) | C2 |

### MVP0 — Quality gates (DRAFT)

| Row ID | Item | Detail | Epic ID | Story ID | Jira key | Constraints | Mismatch / note |
|---|---|---|---|---|---|---|---|
| QG-01 | Backward compatibility | expand/contract migrations; /v1 additive only; tolerant readers ignore unknown fields; defaults for hidden/new fields; event schema BACKWARD compat in registry | EP-MVP0-01, EP-MVP0-04 | ST-MVP0-01-01, ST-MVP0-04-02 | [SCRUM-19](https://rozvaa.atlassian.net/browse/SCRUM-19), [SCRUM-21](https://rozvaa.atlassian.net/browse/SCRUM-21) | C5, C10 | proposed |
| QG-02 | Regression + contract tests in CI | full regression suite + Pact/OpenAPI contract tests block merge | EP-MVP0-01 | ST-MVP0-01-03 | [SCRUM-24](https://rozvaa.atlassian.net/browse/SCRUM-24) | C11 | proposed |
| QG-03 | Lint | dead code, unused vars/imports fail build (ESLint/ts-prune or equivalent) | NO STORY | NO STORY | — | C11 | proposed |
| QG-04 | Typed exceptions | domain exceptions typed and mapped 1:1 to error codes (envelope per MM-18) | EP-MVP0-01 | ST-MVP0-01-03 | [SCRUM-24](https://rozvaa.atlassian.net/browse/SCRUM-24) | C12 | proposed |
| QG-05 | Flag per story | each story ships behind a flag with OFF/ON behaviour; OFF never breaks related data (section 5 col N) | EP-MVP0-03 | ST-MVP0-03-01 | [SCRUM-25](https://rozvaa.atlassian.net/browse/SCRUM-25) | C9 | proposed |
| QG-06 | OWASP Top 10 / SAST / DAST / deps | SAST + dependency scan 0 high/critical; DAST (ZAP) on staging; OWASP checklist per endpoint | EP-MVP0-05 | ST-MVP0-05-01 | [SCRUM-26](https://rozvaa.atlassian.net/browse/SCRUM-26) | C8 | proposed |
| QG-07 | Authz tests | per endpoint: each role allowed/denied matrix + cross-tenant (TST-01) | EP-MVP0-02 | ST-MVP0-02-02 | [SCRUM-32](https://rozvaa.atlassian.net/browse/SCRUM-32) | C1, C13 | proposed |
| QG-08 | Leak tests | memory/connection/handle leak checks; tenant-leak (TST-01) | EP-MVP0-01 | ST-MVP0-01-02 | [SCRUM-23](https://rozvaa.atlassian.net/browse/SCRUM-23) | C1, C11 | proposed |
| QG-09 | Soak tests | 8 h @ 50 rps per service in staging; no memory growth, p95 held | NO STORY | NO STORY | — | C12 | proposed |
| QG-10 | Resilience tests | fault injection per RES row: timeout, breaker open, fallback verified | NO STORY | NO STORY | — | C6 | proposed |

### MVP0 — Open decisions (UNCONFIRMED) (DRAFT)

| Row ID | Item | Detail | Epic ID | Story ID | Jira key | Mismatch / note |
|---|---|---|---|---|---|---|
| OD1 | Cloud SQL vs Supabase — UNCONFIRMED | Architect Doc assumes Cloud SQL; ST-MVP0-01-01 still says either | — | NO STORY | — | UNCONFIRMED |
| OD2 | Outbox in MVP0 W3 (A5) — UNCONFIRMED | Doc agrees; Epic 04 W2-W3 | — | NO STORY | — | UNCONFIRMED |
| OD3 | Observability: Grafana; Sentry — UNCONFIRMED | Doc drops Sentry; Epic 06 still lists Sentry | — | NO STORY | — | UNCONFIRMED |
| OD4 | server/ Express app = prototype — UNCONFIRMED |  | — | NO STORY | — | UNCONFIRMED |
| OD5 | Payroll out of MVP1 — UNCONFIRMED |  | — | NO STORY | — | UNCONFIRMED |
| OD6 | Event bus — UNCONFIRMED | Originally proposed Pub/Sub; Doc rev 2 chooses Kafka (Confluent); Epic 04 says BullMQ | — | NO STORY | — | UNCONFIRMED |
| OD7 | Login: Identity Platform vs custom OTP API — UNCONFIRMED | Story 02-01 defines custom OTP endpoints | — | NO STORY | — | UNCONFIRMED |
| OD8 | Role names — UNCONFIRMED | See RBAC-01 / MM-09 | — | NO STORY | — | UNCONFIRMED |
| OD9 | Task M6 referenced but missing — UNCONFIRMED |  | — | NO STORY | — | UNCONFIRMED |

### MVP0 — Mismatches (flagged, not resolved) (DRAFT)

| Row ID | Item | Detail | Epic ID | Story ID | Jira key | Constraints | Mismatch / note |
|---|---|---|---|---|---|---|---|
| MM-01 | Path shape | Stories: /v1/<resource>; Doc gateway: /v1/<service>/* | EP-MVP0-01 | ST-MVP0-01-03 | [SCRUM-24](https://rozvaa.atlassian.net/browse/SCRUM-24) |  | UNRESOLVED |
| MM-02 | Company vs tenant | Story POST /v1/companies; Doc POST /v1/tenants + /v1/tenants/{id}/route | EP-MVP0-01 | ST-MVP0-01-03 | [SCRUM-24](https://rozvaa.atlassian.net/browse/SCRUM-24) |  | UNRESOLVED |
| MM-03 | Flag evaluate | Story POST /v1/flags/evaluate; Doc GET /v1/flags/evaluate; Arch tab /v1/evaluate | EP-MVP0-03 | ST-MVP0-03-01 | [SCRUM-25](https://rozvaa.atlassian.net/browse/SCRUM-25) |  | UNRESOLVED |
| MM-04 | WhatsApp webhook | Story POST /webhooks/whatsapp (unversioned, violates C5); Doc /v1/hooks/{provider} | EP-MVP0-11 | ST-MVP0-11-01 | [SCRUM-40](https://rozvaa.atlassian.net/browse/SCRUM-40) |  | UNRESOLVED |
| MM-05 | Auth model | Stories: 'Bearer JWT with tenant_id claim' on every API incl. public OTP; custom OTP API. Doc: login via Identity Platform, services trust gateway-signed context (OD7) | EP-MVP0-02 | ST-MVP0-02-01 | [SCRUM-31](https://rozvaa.atlassian.net/browse/SCRUM-31) |  | UNRESOLVED |
| MM-06 | Money units | Story amounts (target_amount 500000, outstanding, glance, amount_gt 50000) have no unit; C15 requires integer paise → rename *_paise or state rupees→paise | EP-MVP0-13, EP-MVP0-09, EP-MVP0-08 | ST-MVP0-13-01, ST-MVP0-09-03, ST-MVP0-08-01 | [SCRUM-42](https://rozvaa.atlassian.net/browse/SCRUM-42), [SCRUM-48](https://rozvaa.atlassian.net/browse/SCRUM-48), [SCRUM-37](https://rozvaa.atlassian.net/browse/SCRUM-37) |  | UNRESOLVED |
| MM-07 | AI chat path | Story /v1/ai/chat; Doc /v1/chat | EP-MVP0-09 | ST-MVP0-09-01 | [SCRUM-38](https://rozvaa.atlassian.net/browse/SCRUM-38) |  | UNRESOLVED |
| MM-08 | Approvals | Story /v1/approval-rules + /v1/approvals/{id}/decision; Doc /v1/proposals/{id}/approve; service owner (ai-agents vs separate) unstated in Doc for rules | EP-MVP0-08 | ST-MVP0-08-01 | [SCRUM-37](https://rozvaa.atlassian.net/browse/SCRUM-37) |  | UNRESOLVED |
| MM-09 | Roles/permissions | Story 'inventory:read' vs Doc 'module.resource.action'; Epic 02 role list vs ADMIN/STORE/PLATFORM_ADMIN used in stories | EP-MVP0-02 | ST-MVP0-02-02 | [SCRUM-32](https://rozvaa.atlassian.net/browse/SCRUM-32) |  | UNRESOLVED |
| MM-10 | Invitations | Story /v1/invitations; Doc lists /v1/users | EP-MVP0-02 | ST-MVP0-02-03 | [SCRUM-33](https://rozvaa.atlassian.net/browse/SCRUM-33) |  | UNRESOLVED |
| MM-11 | Tickets vs threads | Story /v1/tickets and /v1/tickets/{id}/draft; Doc notifications /v1/threads; AI drafting belongs to ai-agents | EP-MVP0-11 | ST-MVP0-11-01, ST-MVP0-11-02 | [SCRUM-40](https://rozvaa.atlassian.net/browse/SCRUM-40), [SCRUM-50](https://rozvaa.atlassian.net/browse/SCRUM-50) |  | UNRESOLVED |
| MM-12 | Kill switch | Story effective_within_s 30; Epic 03 DoD <60 s; earlier MVP0 sheet <10 s. scope per-tenant needs a target tenant from a platform admin → C2 exception needed | EP-MVP0-03 | ST-MVP0-03-02 | [SCRUM-34](https://rozvaa.atlassian.net/browse/SCRUM-34) |  | UNRESOLVED |
| MM-13 | Unowned resources | Imports, document series, home summary, compliance calendar not assigned to a service in Doc; placed provisionally (identity-tenancy, billing, ai-agents, billing) | EP-MVP0-01, EP-MVP0-09 | ST-MVP0-01-07, ST-MVP0-01-05, ST-MVP0-09-03 | [SCRUM-30](https://rozvaa.atlassian.net/browse/SCRUM-30), [SCRUM-29](https://rozvaa.atlassian.net/browse/SCRUM-29), [SCRUM-48](https://rozvaa.atlassian.net/browse/SCRUM-48) |  | UNRESOLVED |
| MM-14 | Connections | Doc: adapters + creds inside owning services; story: one central /v1/connections hub | EP-MVP0-14 | ST-MVP0-14-01 | [SCRUM-54](https://rozvaa.atlassian.net/browse/SCRUM-54) |  | UNRESOLVED |
| MM-15 | BOM check | No BOM tables in Doc inventory; story returns po_draft_id synchronously (cross-service) vs async-first C6 | EP-MVP0-12 | ST-MVP0-12-02 | [SCRUM-51](https://rozvaa.atlassian.net/browse/SCRUM-51) |  | UNRESOLVED |
| MM-16 | Sentry / BullMQ / Supabase | ST-MVP0-06-01 + Epic 06 say Sentry (Doc dropped it); Epic 04 says BullMQ then Pub/Sub (Doc: Kafka); ST-MVP0-01-01 says Cloud SQL/Supabase | EP-MVP0-06, EP-MVP0-04, EP-MVP0-01 | ST-MVP0-06-01, ST-MVP0-04-01, ST-MVP0-01-01 | [SCRUM-22](https://rozvaa.atlassian.net/browse/SCRUM-22), [SCRUM-20](https://rozvaa.atlassian.net/browse/SCRUM-20), [SCRUM-19](https://rozvaa.atlassian.net/browse/SCRUM-19) |  | UNRESOLVED |
| MM-17 | Audit path | Arch tab /v1/audit vs Doc+story /v1/audit-events; platform audit log is cross-tenant (C1 allowlist) | EP-MVP0-05 | ST-MVP0-05-03 | [SCRUM-35](https://rozvaa.atlassian.net/browse/SCRUM-35) |  | UNRESOLVED |
| MM-18 | Error envelope vs problem+json | Story contracts use {"error":{code,message,field,trace_id}}; engineering standard says typed exceptions mapped to problem+json codes | EP-MVP0-01, EP-MVP0-04 | ST-MVP0-01-03, ST-MVP0-04-02 | [SCRUM-24](https://rozvaa.atlassian.net/browse/SCRUM-24), [SCRUM-21](https://rozvaa.atlassian.net/browse/SCRUM-21) | C12 | UNRESOLVED |
| MM-19 | p95 target | Story DoD p95 <300 ms @50 rps per API; Doc gateway p95 <500 ms | EP-MVP0-07 | ST-MVP0-07-02 | [SCRUM-60](https://rozvaa.atlassian.net/browse/SCRUM-60) | C12 | UNRESOLVED |

### MVP0 — Coverage gaps (DRAFT)

| Row ID | Item | Detail | Epic ID | Story ID | Jira key |
|---|---|---|---|---|---|
| GAP-S3 | Story partly uncovered | ST-MVP0-13-03 returns task_id but no service owns tasks | EP-MVP0-13 | ST-MVP0-13-03 | [SCRUM-53](https://rozvaa.atlassian.net/browse/SCRUM-53) |
| GAP-S4 | Story partly uncovered | ST-MVP0-11-03 'Raise CAPA' has no API in any story and no owning service | EP-MVP0-11 | ST-MVP0-11-03 | [SCRUM-64](https://rozvaa.atlassian.net/browse/SCRUM-64) |
| GAP-S5 | Story partly uncovered | ST-MVP0-14-02 WABA/DLT registration is ops work; only templates table designed | EP-MVP0-14 | ST-MVP0-14-02 | [SCRUM-69](https://rozvaa.atlassian.net/browse/SCRUM-69) |
| GAP-S6 | Story partly uncovered | ST-MVP0-12-03 counter-offer request/response not specified | EP-MVP0-12 | ST-MVP0-12-03 | [SCRUM-65](https://rozvaa.atlassian.net/browse/SCRUM-65) |
| GAP-D | Design rows with no story | API gateway; tenant switch / token refresh; invite-accept; approvals list; invoices + invoice.issued (billing); payments + payment.received; customers CRUD (sales-crm); documents upload API; entitlements API; tenant_catalog admin | NO STORY | NO STORY | — |

## MVP1

### MVP1 — Resilience (DRAFT)

| Row ID | Item | Detail | Epic ID | Story ID | Jira key | Constraints | Mismatch / note |
|---|---|---|---|---|---|---|---|
| M1-RES-INT | internal service calls | 3 s timeout, bulkhead per downstream service (all MVP1 stories) | EP-MVP1-01..20 | all ST-MVP1 | [SCRUM-90](https://rozvaa.atlassian.net/browse/SCRUM-90)..143 | C6 | story |

### MVP1 — Tests (DRAFT)

| Row ID | Item | Detail | Epic ID | Story ID | Jira key | Constraints |
|---|---|---|---|---|---|---|
| M1-TST-01 | CA cross-tenant leak suite (membership-scoped RLS) | from story | EP-MVP1-20 | ST-MVP1-20-01 | [SCRUM-132](https://rozvaa.atlassian.net/browse/SCRUM-132) (epic [SCRUM-89](https://rozvaa.atlassian.net/browse/SCRUM-89)) | C1, C11 |
| M1-TST-02 | GST golden tests CA-signed (Q1) | from story | EP-MVP1-03 | ST-MVP1-03-03 | [SCRUM-133](https://rozvaa.atlassian.net/browse/SCRUM-133) (epic [SCRUM-72](https://rozvaa.atlassian.net/browse/SCRUM-72)) | C1, C11 |
| M1-TST-03 | Tax engine golden tests (money.tax) | from story | EP-MVP1-01 | ST-MVP1-01-03 | [SCRUM-95](https://rozvaa.atlassian.net/browse/SCRUM-95) (epic [SCRUM-70](https://rozvaa.atlassian.net/browse/SCRUM-70)) | C1, C11 |
| M1-TST-04 | Narrative grounding validator tests | from story | EP-MVP1-09 | ST-MVP1-09-02 | [SCRUM-119](https://rozvaa.atlassian.net/browse/SCRUM-119) (epic [SCRUM-78](https://rozvaa.atlassian.net/browse/SCRUM-78)) | C1, C11 |
| M1-TST-05 | ASR WER eval harness in CI | from story | EP-MVP1-19 | ST-MVP1-19-02 | [SCRUM-131](https://rozvaa.atlassian.net/browse/SCRUM-131) (epic [SCRUM-88](https://rozvaa.atlassian.net/browse/SCRUM-88)) | C1, C11 |
| M1-TST-06 | DR drill + VAPT (no infra provisioned by this task) | from story | EP-MVP1-15 | ST-MVP1-15-02 | [SCRUM-130](https://rozvaa.atlassian.net/browse/SCRUM-130) (epic [SCRUM-84](https://rozvaa.atlassian.net/browse/SCRUM-84)) | C1, C11 |

### MVP1 — Mismatches (flagged, not resolved) (DRAFT)

| Row ID | Item | Detail | Epic ID | Story ID | Jira key | Mismatch / note |
|---|---|---|---|---|---|---|
| M1-MM-01 | Money type | MVP1 field specs use amount_inr decimal(18,2); C15/Doc require integer paise | EP-MVP1-06 | ST-MVP1-06-01 | [SCRUM-94](https://rozvaa.atlassian.net/browse/SCRUM-94) (epic [SCRUM-75](https://rozvaa.atlassian.net/browse/SCRUM-75)) | UNRESOLVED |
| M1-MM-02 | p95 target | MVP1 engineering notes: ASSUMPTION p95 <500 ms (SLO-01 open); MVP0 DoD p95 <300 ms @50 rps | EP-MVP1-01 | ST-MVP1-01-01 | [SCRUM-91](https://rozvaa.atlassian.net/browse/SCRUM-91) (epic [SCRUM-70](https://rozvaa.atlassian.net/browse/SCRUM-70)) | UNRESOLVED |
| M1-MM-03 | Event names | Stories PascalCase (LeadCreated, OrderConfirmed…) vs Doc <domain>.<event>.v1 | EP-MVP1-02 | ST-MVP1-02-02 | [SCRUM-107](https://rozvaa.atlassian.net/browse/SCRUM-107) (epic [SCRUM-71](https://rozvaa.atlassian.net/browse/SCRUM-71)) | UNRESOLVED |
| M1-MM-04 | SaaS billing owner | Plans/subscriptions/metering/dunning/tenant billing (EP-MVP1-11/12) have no service in Doc; placed provisionally in identity-tenancy | EP-MVP1-11 | ST-MVP1-11-01 | [SCRUM-102](https://rozvaa.atlassian.net/browse/SCRUM-102) (epic [SCRUM-80](https://rozvaa.atlassian.net/browse/SCRUM-80)) | UNRESOLVED |
| M1-MM-05 | Unowned modules | Reports (09-01), GST returns, bank rec, TDS, three-way match, DPDP privacy, sandboxes have no explicit Doc service; placed provisionally | EP-MVP1-09 | ST-MVP1-09-01 | [SCRUM-109](https://rozvaa.atlassian.net/browse/SCRUM-109) (epic [SCRUM-78](https://rozvaa.atlassian.net/browse/SCRUM-78)) | UNRESOLVED |
| M1-MM-06 | Unversioned paths | POST /webhooks/razorpay, /webhooks/whatsapp, /internal/meters/events outside /v1 (C5) | EP-MVP1-11 | ST-MVP1-11-02 | [SCRUM-103](https://rozvaa.atlassian.net/browse/SCRUM-103) (epic [SCRUM-80](https://rozvaa.atlassian.net/browse/SCRUM-80)) | UNRESOLVED |
| M1-MM-07 | tenant id in URL | POST /v1/tenants/{id}/ca-invitations puts tenant id in URL (rule: none in URL/header/body) | EP-MVP1-20 | ST-MVP1-20-01 | [SCRUM-132](https://rozvaa.atlassian.net/browse/SCRUM-132) (epic [SCRUM-89](https://rozvaa.atlassian.net/browse/SCRUM-89)) | UNRESOLVED |
| M1-MM-08 | Cross-tenant CA access | RLS by membership spans tenants — needs C1 exception design | EP-MVP1-20 | ST-MVP1-20-01 | [SCRUM-132](https://rozvaa.atlassian.net/browse/SCRUM-132) (epic [SCRUM-89](https://rozvaa.atlassian.net/browse/SCRUM-89)) | UNRESOLVED |
| M1-MM-09 | Flag key format | Stories: sell.crm.01_01 style; MVP0 tab proposed home.chat etc.; Arch tab stage model | EP-MVP1-01 | ST-MVP1-01-01 | [SCRUM-91](https://rozvaa.atlassian.net/browse/SCRUM-91) (epic [SCRUM-70](https://rozvaa.atlassian.net/browse/SCRUM-70)) | UNRESOLVED |
| M1-MM-10 | Error envelope | MVP1 uses "details"; MVP0 uses "field"; standard says problem+json (MM-18) | EP-MVP1-01 | ST-MVP1-01-01 | [SCRUM-91](https://rozvaa.atlassian.net/browse/SCRUM-91) (epic [SCRUM-70](https://rozvaa.atlassian.net/browse/SCRUM-70)) | UNRESOLVED |
| M1-MM-11 | Event bus / queue | Story chaos sections: Postgres outbox → Pub/Sub + BullMQ on Valkey, "not Kafka"; Doc rev 2: Kafka (OD6) | EP-MVP1-13 | ST-MVP1-13-02 | [SCRUM-105](https://rozvaa.atlassian.net/browse/SCRUM-105) (epic [SCRUM-82](https://rozvaa.atlassian.net/browse/SCRUM-82)) | UNRESOLVED |
| M1-MM-12 | Tally connector | Desktop connector + /v1/tally/sync/batches with connector token (not gateway JWT); adapter placement (MM-14) | EP-MVP1-07 | ST-MVP1-07-01 | [SCRUM-90](https://rozvaa.atlassian.net/browse/SCRUM-90) (epic [SCRUM-76](https://rozvaa.atlassian.net/browse/SCRUM-76)) | UNRESOLVED |
| M1-MM-13 | Statutory names | Form 140 (formerly 26Q), Form 131, IT Act 2025 effective dates — not verified | EP-MVP1-08 | ST-MVP1-08-02 | [SCRUM-118](https://rozvaa.atlassian.net/browse/SCRUM-118) (epic [SCRUM-77](https://rozvaa.atlassian.net/browse/SCRUM-77)) | UNRESOLVED |
| M1-MM-14 | Synchronous cross-service | Quote convert→order, order confirm credit check, three-way match read across sales/purchase/billing synchronously vs async-first C6 | EP-MVP1-18 | ST-MVP1-18-01 | [SCRUM-113](https://rozvaa.atlassian.net/browse/SCRUM-113) (epic [SCRUM-87](https://rozvaa.atlassian.net/browse/SCRUM-87)) | UNRESOLVED |
| M1-MM-15 | Missing epic links | Jira parent epic not set on 33 of 54 MVP1 stories (e.g. [SCRUM-90](https://rozvaa.atlassian.net/browse/SCRUM-90)..95, 98, 99, 101-103, 108, 112, 113…); epic inferred from ST id | EP-MVP1-07 | ST-MVP1-07-01 | [SCRUM-90](https://rozvaa.atlassian.net/browse/SCRUM-90) (epic [SCRUM-76](https://rozvaa.atlassian.net/browse/SCRUM-76)) | UNRESOLVED |

### MVP1 — Coverage gaps (DRAFT)

| Row ID | Item | Detail | Epic ID | Story ID | Jira key |
|---|---|---|---|---|---|
| M1-GAP-01 | No API contract | UI/infra stories 104,105,130,131 have no endpoint; [SCRUM-134](https://rozvaa.atlassian.net/browse/SCRUM-134) go-live wizard calls unspecified | EP-MVP1-07 | ST-MVP1-07-02 | [SCRUM-134](https://rozvaa.atlassian.net/browse/SCRUM-134) (epic [SCRUM-76](https://rozvaa.atlassian.net/browse/SCRUM-76)) |
| M1-GAP-02 | Calls unspecified | UI stories list screens but not endpoints for orders docs, match workbench push, reports hub | EP-MVP1-02 | ST-MVP1-02-03 | [SCRUM-125](https://rozvaa.atlassian.net/browse/SCRUM-125) (epic [SCRUM-71](https://rozvaa.atlassian.net/browse/SCRUM-71)) |
| M1-GAP-03 | Events consumers | No story defines consumer for po.approved, einvoice.irn_generated | EP-MVP1-04 | ST-MVP1-04-01 | [SCRUM-98](https://rozvaa.atlassian.net/browse/SCRUM-98) (epic [SCRUM-73](https://rozvaa.atlassian.net/browse/SCRUM-73)) |
| M1-GAP-04 | Design rows with no story | GRN (goods receipt) table/API needed by three-way match; customer/supplier master in MVP1; sales invoice create API (IRN assumes invoices exist) | EP-MVP1-18 | ST-MVP1-18-01 | [SCRUM-113](https://rozvaa.atlassian.net/browse/SCRUM-113) (epic [SCRUM-87](https://rozvaa.atlassian.net/browse/SCRUM-87)) |
