# collections-payments — MVP0/MVP1 design (DRAFT)

> **DRAFT — not frozen.** Design only: no OpenAPI, nothing provisioned. Open decisions OD1–OD9 are **UNCONFIRMED**. Mismatches with the Architect Doc are flagged, not resolved.

Sources: [Sheet tab 'MVP0-1 Design'](https://docs.google.com/spreadsheets/d/18p-aJ0QeW6L3SJ-CNSrNCjwKWBpVdkQAfkzeKOg40Oc/edit#gid=119904232) · [BOS Architect's Doc](https://docs.google.com/document/d/1sKoivvz3JLkKPI1Jta0i6XWnWuW57-rSwWICmm1w9VM/edit) · Jira SCRUM (MVP0 + MVP1 epics SCRUM-70..89, stories SCRUM-90..143). Generated 31 Oct 2026 IST.

Conventions, error/idempotency rules, quality gates, open decisions, mismatches and gaps: see [README](README.md). Rows: MVP0 14, MVP1 18.

## MVP0

### MVP0 — Data model (DRAFT)

| Row ID | Item | Detail | Epic ID | Story ID | Jira key | Constraints | Mismatch / note |
|---|---|---|---|---|---|---|---|
| TBL-CP-01 | receivables (read model) | tenant_id, invoice_id, party_id, outstanding_paise, due_date, oldest_days, bucket — built from invoice.issued / payment.received events | EP-MVP0-13 | ST-MVP0-13-01 | [SCRUM-42](https://rozvaa.atlassian.net/browse/SCRUM-42) | C1, C4, C6, C15 |  |
| TBL-CP-02 | collection_sprints, sprint_items | tenant_id, id, target_paise, ends_on; items: party_id, outstanding_paise | EP-MVP0-13 | ST-MVP0-13-01 | [SCRUM-42](https://rozvaa.atlassian.net/browse/SCRUM-42) | C1, C15 |  |
| TBL-CP-03 | follow_ups | tenant_id, id, party_id, channel, template, schedule_at, status(SCHEDULED\|SENT\|FAILED) | EP-MVP0-13 | ST-MVP0-13-02 | [SCRUM-52](https://rozvaa.atlassian.net/browse/SCRUM-52) | C1, C6 |  |
| TBL-CP-04 | disputes | tenant_id, id, invoice_id, reason, note, task_id | EP-MVP0-13 | ST-MVP0-13-03 | [SCRUM-53](https://rozvaa.atlassian.net/browse/SCRUM-53) | C1 | No tasks service; see GAP-S3. |
| TBL-CP-05 | payments | tenant_id, id, invoice_id, amount_paise, method, ref — no card data | NO STORY | NO STORY | — | C1, C15 | Epic 13 DoD says payments recorded; no API story (GAP-D). |

### MVP0 — API contracts (DRAFT)

| Row ID | Item | Detail | Role / permission | Idempotency-Key | Emits / consumes | Epic ID | Story ID | Jira key | Constraints | Mismatch / note | p95 latency target | Payload limit |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| API-CP-01 | GET /v1/receivables?bucket=&cursor= | list {items,next_cursor} | ACCOUNTANT/OWNER | n/a | — | EP-MVP0-13 | ST-MVP0-13-01 | [SCRUM-42](https://rozvaa.atlassian.net/browse/SCRUM-42) | C5, C15 |  | <300 ms @50 rps (story DoD) | 1 MB (proposed) |
| API-CP-02 | POST /v1/collection-sprints | {party_ids, target_amount, ends_on} → {id, items[{party_id, outstanding, oldest_days}]}. PARTY_NOT_FOUND 404 | ACCOUNTANT/OWNER | Yes | collections.sprint.started.v1 | EP-MVP0-13 | ST-MVP0-13-01 | [SCRUM-42](https://rozvaa.atlassian.net/browse/SCRUM-42) | C15 | target_amount/outstanding unit unstated vs paise (MM-06). | <300 ms @50 rps (story DoD) | 1 MB (proposed) |
| API-CP-03 | POST /v1/follow-ups | {party_id, channel, template, schedule} → {id, status SCHEDULED}. NO_OPTIN 422, TEMPLATE_NOT_APPROVED 422, OUTSIDE_WINDOW 422 | ACCOUNTANT | Yes | collections.followup.scheduled.v1 → notifications sends | EP-MVP0-13 | ST-MVP0-13-02 | [SCRUM-52](https://rozvaa.atlassian.net/browse/SCRUM-52) | C6, C7, C13 |  | <300 ms @50 rps (story DoD) | 1 MB (proposed) |
| API-CP-04 | POST /v1/disputes | {invoice_id, reason, note} → {id, task_id} | ACCOUNTANT/SALES | Yes | collections.dispute.raised.v1 | EP-MVP0-13 | ST-MVP0-13-03 | [SCRUM-53](https://rozvaa.atlassian.net/browse/SCRUM-53) | C4 | task_id has no owning service (GAP-S3). | <300 ms @50 rps (story DoD) | 1 MB (proposed) |

### MVP0 — Events (DRAFT)

| Row ID | Item | Detail | Emits / consumes | Epic ID | Story ID | Jira key | Constraints | Mismatch / note |
|---|---|---|---|---|---|---|---|---|
| EVT-06 | collections.followup.scheduled.v1 | consumers: notifications, audit; data: follow_up_id, party_id, channel, template, schedule_at | producer: collections-payments | EP-MVP0-13 | ST-MVP0-13-02 | [SCRUM-52](https://rozvaa.atlassian.net/browse/SCRUM-52) | C4, C6, C12 |  |
| EVT-07 | collections.dispute.raised.v1 | consumers: audit, (tasks owner TBD); data: dispute_id, invoice_id, reason | producer: collections-payments | EP-MVP0-13 | ST-MVP0-13-03 | [SCRUM-53](https://rozvaa.atlassian.net/browse/SCRUM-53) | C4, C6, C12 |  |
| EVT-18 | payment.received.v1 | consumers: billing, sales-crm, ai-agents (home glance); data: payment_id, invoice_id, amount_paise | producer: collections-payments | NO STORY | NO STORY | — | C4, C6, C12 | Needed by MVP0 receivables but no MVP0 story produces it. |

### MVP0 — Resilience (DRAFT)

| Row ID | Item | Detail | Epic ID | Story ID | Jira key | Constraints | Mismatch / note |
|---|---|---|---|---|---|---|---|
| RES-CP-01 | → billing (invoices) read | timeout 2 s; breaker; fallback: serve last receivables snapshot with as_of | EP-MVP0-13 | ST-MVP0-13-01 | [SCRUM-42](https://rozvaa.atlassian.net/browse/SCRUM-42) | C6, C12 | proposed |

### MVP0 — Observability (DRAFT)

| Row ID | Item | Detail | Epic ID | Story ID | Jira key | Constraints | Mismatch / note |
|---|---|---|---|---|---|---|---|
| OBS-COLLEC | RED + business metrics | RED per endpoint; business: followups_sent, amount_collected_paise, dso_days; JSON logs PII-masked (TST-06) with trace_id, hashed tenant_id; OTel spans incl. DB/Kafka/outbound; SLO: follow-up dispatch ≥99% (proposed unless story); alerts: burn-rate 2%/1h & 5%/6h, DLQ>0, breaker open; Grafana panel "collections-payments" | EP-MVP0-06 | ST-MVP0-06-01, ST-MVP0-06-02 | [SCRUM-22](https://rozvaa.atlassian.net/browse/SCRUM-22), [SCRUM-28](https://rozvaa.atlassian.net/browse/SCRUM-28) | C12 | proposed |

## MVP1

### MVP1 — Data model (DRAFT)

| Row ID | Item | Detail | Epic ID | Story ID | Jira key | Constraints |
|---|---|---|---|---|---|---|
| M1-TBL-COLL-01 | ageing read model (receivables/payables, msme_class, acceptance_date) | tenant_id + RLS policy; story | EP-MVP1-06 | ST-MVP1-06-01 | [SCRUM-94](https://rozvaa.atlassian.net/browse/SCRUM-94) (epic [SCRUM-75](https://rozvaa.atlassian.net/browse/SCRUM-75)) | C1, C3, C15 |
| M1-TBL-COLL-02 | payments, payment_allocations; cash_runway_snapshots; bank_feed_uploads | tenant_id + RLS policy; FIFO default (story) | EP-MVP1-06 | ST-MVP1-06-02 | [SCRUM-100](https://rozvaa.atlassian.net/browse/SCRUM-100) (epic [SCRUM-75](https://rozvaa.atlassian.net/browse/SCRUM-75)) | C1, C3, C15 |
| M1-TBL-COLL-03 | bank_statements, bank_lines unique (account, bank_txn_id) | tenant_id + RLS policy; story | EP-MVP1-08 | ST-MVP1-08-01 | [SCRUM-101](https://rozvaa.atlassian.net/browse/SCRUM-101) (epic [SCRUM-77](https://rozvaa.atlassian.net/browse/SCRUM-77)) | C1, C3, C15 |
| M1-TBL-COLL-04 | tds_register (versioned by effective date) | tenant_id + RLS policy; story | EP-MVP1-08 | ST-MVP1-08-02 | [SCRUM-118](https://rozvaa.atlassian.net/browse/SCRUM-118) (epic [SCRUM-77](https://rozvaa.atlassian.net/browse/SCRUM-77)) | C1, C3, C15 |

### MVP1 — API contracts (DRAFT)

| Row ID | Item | Detail | Role / permission | Idempotency-Key | Emits / consumes | Epic ID | Story ID | Jira key | Constraints | Mismatch / note | p95 latency target | Payload limit |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| M1-API-06-01 | GET /v1/ageing/receivables?as_of= ; GET /v1/ageing/payables?msme_only= | Req: - Resp: {"as_of":"2026-10-30","buckets":[{"label":"0-30","amount":1250000.00}],"msme_at_risk":[{"party_id":"sup_2","due":"2026-11-26","amount":59400.00}]} Errors: standard envelope | ACCOUNTANT | n/a | outbox events per M1-3 | EP-MVP1-06 | ST-MVP1-06-01 | [SCRUM-94](https://rozvaa.atlassian.net/browse/SCRUM-94) (epic [SCRUM-75](https://rozvaa.atlassian.net/browse/SCRUM-75)) | C2, C5, C6, C12 | blocked by [SCRUM-42](https://rozvaa.atlassian.net/browse/SCRUM-42) | <500 ms (story ASSUMPTION; MVP0 DoD <300 ms — M1-MM-02) | 1 MB JSON (story) |
| M1-API-06-02 | POST /v1/payments ; GET /v1/cash/runway?weeks=13 ; POST /v1/bank-feeds/upload | Req: {"party_id":"p_4","direction":"IN","amount":60000,"mode":"NEFT","utr":"SBIN526303123456","allocations":"AUTO","tds":{"section":"194J-TECH","amount":1200}} Resp: {"id":"pay_1","allocated":[{"invoice_id":"inv_1","amount":50000},{"invoice_id":"inv_2","amount":10000}]} Errors: OVER_ALLOCATION 422, DUPLICATE_UTR 409 | ACCOUNTANT | Yes (POST) | outbox events per M1-3 | EP-MVP1-06 | ST-MVP1-06-02 | [SCRUM-100](https://rozvaa.atlassian.net/browse/SCRUM-100) (epic [SCRUM-75](https://rozvaa.atlassian.net/browse/SCRUM-75)) | C2, C5, C6, C12 | blocked by [SCRUM-94](https://rozvaa.atlassian.net/browse/SCRUM-94), [SCRUM-52](https://rozvaa.atlassian.net/browse/SCRUM-52) | <500 ms (story ASSUMPTION; MVP0 DoD <300 ms — M1-MM-02) | 1 MB JSON (story) |
| M1-API-08-01 | POST /v1/bank/statements ; POST /v1/bank/lines/{id}/match | Req: {"line_id":"bl_1","payment_ids":["pay_1"]} Resp: {"status":"MATCHED"} Errors: MATCH_AMOUNT_MISMATCH 422 | ACCOUNTANT | Yes (POST) | outbox events per M1-3 | EP-MVP1-08 | ST-MVP1-08-01 | [SCRUM-101](https://rozvaa.atlassian.net/browse/SCRUM-101) (epic [SCRUM-77](https://rozvaa.atlassian.net/browse/SCRUM-77)) | C2, C5, C6, C12 |  | <500 ms (story ASSUMPTION; MVP0 DoD <300 ms — M1-MM-02) | 1 MB JSON (story) |
| M1-API-08-02 | GET /v1/tds/register?quarter=Q2&fy=2026-27 ; GET /v1/tds/statements/Q2/export | Req: - Resp: {"quarter":"Q2","form":"140","due":"2026-10-31","sections":[{"section":"194J-TECH","deductees":4,"base":400000.00,"tds":8000.00}]} Errors: FY_NOT_SUPPORTED 422 | ACCOUNTANT/CA | n/a | outbox events per M1-3 | EP-MVP1-08 | ST-MVP1-08-02 | [SCRUM-118](https://rozvaa.atlassian.net/browse/SCRUM-118) (epic [SCRUM-77](https://rozvaa.atlassian.net/browse/SCRUM-77)) | C2, C5, C6, C12 |  | <500 ms (story ASSUMPTION; MVP0 DoD <300 ms — M1-MM-02) | 1 MB JSON (story) |

### MVP1 — Feature flags (with OFF behaviour) (DRAFT)

| Row ID | Item | Detail | Epic ID | Story ID | Jira key | Constraints | Mismatch / note | Flag OFF behaviour |
|---|---|---|---|---|---|---|---|---|
| M1-FLG-06-01 | money.finance.06_01 | Receivables & payables ageing API with MSME due tracking | EP-MVP1-06 | ST-MVP1-06-01 | [SCRUM-94](https://rozvaa.atlassian.net/browse/SCRUM-94) (epic [SCRUM-75](https://rozvaa.atlassian.net/browse/SCRUM-75)) | C9 | key format <module>.<area>.<NN_NN> differs from MVP0 proposed keys — M1-MM-09 | OFF: hidden / 404 FEATURE_DISABLED; data readable, no writes lost (story) |
| M1-FLG-06-02 | money.finance.06_02 | Payment recording, reminders, cash runway & bank feed API | EP-MVP1-06 | ST-MVP1-06-02 | [SCRUM-100](https://rozvaa.atlassian.net/browse/SCRUM-100) (epic [SCRUM-75](https://rozvaa.atlassian.net/browse/SCRUM-75)) | C9 | key format <module>.<area>.<NN_NN> differs from MVP0 proposed keys — M1-MM-09 | OFF: hidden / 404 FEATURE_DISABLED; data readable, no writes lost (story) |
| M1-FLG-08-01 | money.bank.08_01 | Bank reconciliation API | EP-MVP1-08 | ST-MVP1-08-01 | [SCRUM-101](https://rozvaa.atlassian.net/browse/SCRUM-101) (epic [SCRUM-77](https://rozvaa.atlassian.net/browse/SCRUM-77)) | C9 | key format <module>.<area>.<NN_NN> differs from MVP0 proposed keys — M1-MM-09 | OFF: hidden / 404 FEATURE_DISABLED; data readable, no writes lost (story) |
| M1-FLG-08-02 | money.bank.08_02 | TDS register & Form 140 (formerly 26Q) summary API | EP-MVP1-08 | ST-MVP1-08-02 | [SCRUM-118](https://rozvaa.atlassian.net/browse/SCRUM-118) (epic [SCRUM-77](https://rozvaa.atlassian.net/browse/SCRUM-77)) | C9 | key format <module>.<area>.<NN_NN> differs from MVP0 proposed keys — M1-MM-09 | OFF: hidden / 404 FEATURE_DISABLED; data readable, no writes lost (story) |
| M1-FLG-06-03 | money.finance.06_03 | Receivables, payables, runway and payment screens | EP-MVP1-06 | ST-MVP1-06-03 | [SCRUM-116](https://rozvaa.atlassian.net/browse/SCRUM-116) (epic [SCRUM-75](https://rozvaa.atlassian.net/browse/SCRUM-75)) | C9 | key format <module>.<area>.<NN_NN> differs from MVP0 proposed keys — M1-MM-09 | OFF: hidden / 404 FEATURE_DISABLED; data readable, no writes lost (story) |
| M1-FLG-08-03 | money.bank.08_03 | Bank reconciliation and TDS screens | EP-MVP1-08 | ST-MVP1-08-03 | [SCRUM-135](https://rozvaa.atlassian.net/browse/SCRUM-135) (epic [SCRUM-77](https://rozvaa.atlassian.net/browse/SCRUM-77)) | C9 | key format <module>.<area>.<NN_NN> differs from MVP0 proposed keys — M1-MM-09 | OFF: hidden / 404 FEATURE_DISABLED; data readable, no writes lost (story) |

### MVP1 — Resilience (DRAFT)

| Row ID | Item | Detail | Epic ID | Story ID | Jira key | Constraints | Mismatch / note |
|---|---|---|---|---|---|---|---|
| M1-RES-06-02 | cb.messaging_gateway_whatsa | timeout 10 s external; breaker 50%/20 calls, half-open 30 s; bulkhead 10/tenant; fallback: degrade to manual / template text, never block the core flow | EP-MVP1-06 | ST-MVP1-06-02 | [SCRUM-100](https://rozvaa.atlassian.net/browse/SCRUM-100) (epic [SCRUM-75](https://rozvaa.atlassian.net/browse/SCRUM-75)) | C6, C12 | story |

### MVP1 — Observability (DRAFT)

| Row ID | Item | Detail | Epic ID | Story ID | Jira key | Constraints | Mismatch / note |
|---|---|---|---|---|---|---|---|
| M1-OBS-COLLEC | business metrics: bos_msme_payables_at_risk_inr, bos_cash_runway_weeks, bos_bank_auto_match_ratio | RED + logs + spans + SLO + alerts + Grafana panel per story (values above) | EP-MVP1-06, EP-MVP1-08 | ST-MVP1-06-01, ST-MVP1-06-02, ST-MVP1-08-01, ST-MVP1-08-02, ST-MVP1-06-03, ST-MVP1-08-03 | [SCRUM-94](https://rozvaa.atlassian.net/browse/SCRUM-94), [SCRUM-100](https://rozvaa.atlassian.net/browse/SCRUM-100), [SCRUM-101](https://rozvaa.atlassian.net/browse/SCRUM-101), [SCRUM-118](https://rozvaa.atlassian.net/browse/SCRUM-118), [SCRUM-116](https://rozvaa.atlassian.net/browse/SCRUM-116), [SCRUM-135](https://rozvaa.atlassian.net/browse/SCRUM-135) | C12 | story |

### MVP1 — UI → API (DRAFT)

| Row ID | Item | Detail | Epic ID | Story ID | Jira key | Constraints | Mismatch / note |
|---|---|---|---|---|---|---|---|
| M1-UI-06-03 | Receivables, payables, runway and payment screens | Money MFE. | EP-MVP1-06 | ST-MVP1-06-03 | [SCRUM-116](https://rozvaa.atlassian.net/browse/SCRUM-116) (epic [SCRUM-75](https://rozvaa.atlassian.net/browse/SCRUM-75)) | C9 |  |
| M1-UI-08-03 | Bank reconciliation and TDS screens | Money MFE. | EP-MVP1-08 | ST-MVP1-08-03 | [SCRUM-135](https://rozvaa.atlassian.net/browse/SCRUM-135) (epic [SCRUM-77](https://rozvaa.atlassian.net/browse/SCRUM-77)) | C9 | blocked by [SCRUM-101](https://rozvaa.atlassian.net/browse/SCRUM-101), [SCRUM-118](https://rozvaa.atlassian.net/browse/SCRUM-118) |
