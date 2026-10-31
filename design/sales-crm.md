# sales-crm — MVP0/MVP1 design (DRAFT)

> **DRAFT — not frozen.** Design only: no OpenAPI, nothing provisioned. Open decisions OD1–OD9 are **UNCONFIRMED**. Mismatches with the Architect Doc are flagged, not resolved.

Sources: [Sheet tab 'MVP0-1 Design'](https://docs.google.com/spreadsheets/d/18p-aJ0QeW6L3SJ-CNSrNCjwKWBpVdkQAfkzeKOg40Oc/edit#gid=119904232) · [BOS Architect's Doc](https://docs.google.com/document/d/1sKoivvz3JLkKPI1Jta0i6XWnWuW57-rSwWICmm1w9VM/edit) · Jira SCRUM (MVP0 + MVP1 epics SCRUM-70..89, stories SCRUM-90..143). Generated 31 Oct 2026 IST.

Conventions, error/idempotency rules, quality gates, open decisions, mismatches and gaps: see [README](README.md). Rows: MVP0 2, MVP1 23.

## MVP0

### MVP0 — Data model (DRAFT)

| Row ID | Item | Detail | Epic ID | Story ID | Jira key | Constraints | Mismatch / note |
|---|---|---|---|---|---|---|---|
| TBL-SC-01 | customers (parties), contacts | tenant_id, id, legal_name, gstin, state_code; contacts with opted_in | NO STORY | NO STORY | — | C1 | Referenced as party_id by 11-01, 13-01; filled by import 01-07; no CRUD story. |

### MVP0 — Observability (DRAFT)

| Row ID | Item | Detail | Epic ID | Story ID | Jira key | Constraints | Mismatch / note |
|---|---|---|---|---|---|---|---|
| OBS-SALES | RED + business metrics | RED per endpoint; business: customers_created; JSON logs PII-masked (TST-06) with trace_id, hashed tenant_id; OTel spans incl. DB/Kafka/outbound; SLO: availability 99.9% (proposed unless story); alerts: burn-rate 2%/1h & 5%/6h, DLQ>0, breaker open; Grafana panel "sales-crm" | EP-MVP0-06 | ST-MVP0-06-01, ST-MVP0-06-02 | [SCRUM-22](https://rozvaa.atlassian.net/browse/SCRUM-22), [SCRUM-28](https://rozvaa.atlassian.net/browse/SCRUM-28) | C12 | proposed |

## MVP1

### MVP1 — Data model (DRAFT)

| Row ID | Item | Detail | Epic ID | Story ID | Jira key | Constraints |
|---|---|---|---|---|---|---|
| M1-TBL-SALE-01 | leads, lead_stages, lead_events | tenant_id + RLS policy; story; dedupe unique (tenant_id, mobile); mobile field-level encrypted | EP-MVP1-01 | ST-MVP1-01-01 | [SCRUM-91](https://rozvaa.atlassian.net/browse/SCRUM-91) (epic [SCRUM-70](https://rozvaa.atlassian.net/browse/SCRUM-70)) | C1, C3, C15 |
| M1-TBL-SALE-02 | follow_ups; customer_360 (read model from events) | tenant_id + RLS policy; read model p95 <400 ms (story ASSUMPTION) | EP-MVP1-01 | ST-MVP1-01-02 | [SCRUM-92](https://rozvaa.atlassian.net/browse/SCRUM-92) (epic [SCRUM-70](https://rozvaa.atlassian.net/browse/SCRUM-70)) | C1, C3, C15 |
| M1-TBL-SALE-03 | quotes, quote_lines, quote_revisions | tenant_id + RLS policy; proposed names; HSN digits per AATO band | EP-MVP1-01 | ST-MVP1-01-03 | [SCRUM-95](https://rozvaa.atlassian.net/browse/SCRUM-95) (epic [SCRUM-70](https://rozvaa.atlassian.net/browse/SCRUM-70)) | C1, C3, C15 |
| M1-TBL-SALE-04 | orders, order_lines (status machine) | tenant_id + RLS policy; story | EP-MVP1-02 | ST-MVP1-02-01 | [SCRUM-96](https://rozvaa.atlassian.net/browse/SCRUM-96) (epic [SCRUM-71](https://rozvaa.atlassian.net/browse/SCRUM-71)) | C1, C3, C15 |
| M1-TBL-SALE-05 | order_requirements (MRP result per order version) | tenant_id + RLS policy; proposed | EP-MVP1-02 | ST-MVP1-02-02 | [SCRUM-107](https://rozvaa.atlassian.net/browse/SCRUM-107) (epic [SCRUM-71](https://rozvaa.atlassian.net/browse/SCRUM-71)) | C1, C3, C15 |

### MVP1 — API contracts (DRAFT)

| Row ID | Item | Detail | Role / permission | Idempotency-Key | Emits / consumes | Epic ID | Story ID | Jira key | Constraints | Mismatch / note | p95 latency target | Payload limit |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| M1-API-01-01 | POST /v1/leads ; PATCH /v1/leads/{id}/stage ; GET /v1/leads?stage=&owner= | Req: {"name":"Shree Ganesh Traders","mobile":"+919876543210","source":"INDIAMART","gstin":"27AAACR5055K1Z7","need":"Brass valves 25mm x 500"} Resp: {"id":"ld_1","stage":"NEW","owner_id":"u_3","created_at":"2026-10-30T10:00:00+05:30"} Errors: LEAD_DUPLICATE 409, STAGE_INVALID 422, STAGE_TRANSITION_BLOCKED 422 (WON requires quote or order) | SALES | Yes (POST) | outbox events per M1-3 | EP-MVP1-01 | ST-MVP1-01-01 | [SCRUM-91](https://rozvaa.atlassian.net/browse/SCRUM-91) (epic [SCRUM-70](https://rozvaa.atlassian.net/browse/SCRUM-70)) | C2, C5, C6, C12 | blocked by [SCRUM-24](https://rozvaa.atlassian.net/browse/SCRUM-24), [SCRUM-32](https://rozvaa.atlassian.net/browse/SCRUM-32) | <500 ms (story ASSUMPTION; MVP0 DoD <300 ms — M1-MM-02) | 1 MB JSON (story) |
| M1-API-01-02 | POST /v1/follow-ups ; GET /v1/parties/{id}/360 | Req: {"party_id":"p_4","type":"CALL","due_at":"2026-11-02T11:00:00+05:30","note":"Share revised quote"} Resp: {"id":"fu_9","status":"OPEN"} Errors: PARTY_NOT_FOUND 404 | SALES | Yes (POST) | outbox events per M1-3 | EP-MVP1-01 | ST-MVP1-01-02 | [SCRUM-92](https://rozvaa.atlassian.net/browse/SCRUM-92) (epic [SCRUM-70](https://rozvaa.atlassian.net/browse/SCRUM-70)) | C2, C5, C6, C12 | blocked by [SCRUM-91](https://rozvaa.atlassian.net/browse/SCRUM-91), [SCRUM-42](https://rozvaa.atlassian.net/browse/SCRUM-42) | <400 ms read (story ASSUMPTION) | 1 MB JSON (story) |
| M1-API-01-03 | POST /v1/quotes ; POST /v1/quotes/{id}/revise ; POST /v1/quotes/{id}/convert | Req: {"party_id":"p_4","place_of_supply":"29","valid_till":"2026-11-15","lines":[{"item_id":"it_1","hsn":"84818090","qty":500,"uom":"NOS","rate":245.5,"gst_rate":18 Resp: {"id":"qt_1","no":"QT/26-27/0042","version":1,"taxable":122750.00,"igst":22095.00,"total":144845.00} Errors: HSN_REQUIRED 422, HSN_DIGITS_INSUFFICIENT 422 (per AATO band), QUOTE_EXPIRED 422 on convert | SALES | Yes (POST) | outbox events per M1-3 | EP-MVP1-01 | ST-MVP1-01-03 | [SCRUM-95](https://rozvaa.atlassian.net/browse/SCRUM-95) (epic [SCRUM-70](https://rozvaa.atlassian.net/browse/SCRUM-70)) | C2, C5, C6, C12 |  | <500 ms (story ASSUMPTION; MVP0 DoD <300 ms — M1-MM-02) | 1 MB JSON (story) |
| M1-API-02-01 | POST /v1/orders ; POST /v1/orders/{id}/confirm ; GET /v1/orders?status= | Req: {"quote_id":"qt_1","commit_date":"2026-11-20","po_ref":"CUST-PO-889"} Resp: {"id":"so_1","no":"SO/26-27/0101","status":"HOLD","credit":{"limit":500000,"outstanding":450000,"order":144845,"reason":"LIMIT_EXCEEDED"},"approval_id":"apr_7"} Errors: CREDIT_HOLD 202 (approval pending), COMMIT_DATE_PAST 422 | SALES (confirm), ACCOUNTANT (override) | Yes (POST) | outbox events per M1-3 | EP-MVP1-02 | ST-MVP1-02-01 | [SCRUM-96](https://rozvaa.atlassian.net/browse/SCRUM-96) (epic [SCRUM-71](https://rozvaa.atlassian.net/browse/SCRUM-71)) | C2, C5, C6, C12 | blocked by [SCRUM-95](https://rozvaa.atlassian.net/browse/SCRUM-95), [SCRUM-42](https://rozvaa.atlassian.net/browse/SCRUM-42), [SCRUM-37](https://rozvaa.atlassian.net/browse/SCRUM-37) | <500 ms (story ASSUMPTION; MVP0 DoD <300 ms — M1-MM-02) | 1 MB JSON (story) |
| M1-API-02-02 | GET /v1/orders/{id}/requirements | Req: - Resp: {"lines":[{"item_id":"it_1","required":500,"available":200,"shortage":300,"pr_id":"pr_3","expected":"2026-11-25"}]} Errors: standard envelope | SALES/PURCHASE | n/a | outbox events per M1-3 | EP-MVP1-02 | ST-MVP1-02-02 | [SCRUM-107](https://rozvaa.atlassian.net/browse/SCRUM-107) (epic [SCRUM-71](https://rozvaa.atlassian.net/browse/SCRUM-71)) | C2, C5, C6, C12 | blocked by [SCRUM-96](https://rozvaa.atlassian.net/browse/SCRUM-96), [SCRUM-93](https://rozvaa.atlassian.net/browse/SCRUM-93) | <500 ms (story ASSUMPTION; MVP0 DoD <300 ms — M1-MM-02) | 1 MB JSON (story) |

### MVP1 — Events (DRAFT)

| Row ID | Item | Detail | Emits / consumes | Epic ID | Story ID | Jira key | Constraints |
|---|---|---|---|---|---|---|---|
| M1-EVT-01 | lead.created.v1 / lead.stage_changed.v1 | story names LeadCreated/LeadStageChanged; consumers: customer_360, ai-agents, audit | producer: sales-crm | EP-MVP1-01 | ST-MVP1-01-01 | [SCRUM-91](https://rozvaa.atlassian.net/browse/SCRUM-91) (epic [SCRUM-70](https://rozvaa.atlassian.net/browse/SCRUM-70)) | C4, C6, C12 |
| M1-EVT-02 | order.credit_checked.v1 | story CreditChecked; consumers: approvals (ai-agents), audit | producer: sales-crm | EP-MVP1-02 | ST-MVP1-02-01 | [SCRUM-96](https://rozvaa.atlassian.net/browse/SCRUM-96) (epic [SCRUM-71](https://rozvaa.atlassian.net/browse/SCRUM-71)) | C4, C6, C12 |
| M1-EVT-03 | order.confirmed.v1 | story OrderConfirmed (MRP trigger); consumers: inventory (reserve), purchase (MRP), billing | producer: sales-crm | EP-MVP1-02 | ST-MVP1-02-02 | [SCRUM-107](https://rozvaa.atlassian.net/browse/SCRUM-107) (epic [SCRUM-71](https://rozvaa.atlassian.net/browse/SCRUM-71)) | C4, C6, C12 |

### MVP1 — Feature flags (with OFF behaviour) (DRAFT)

| Row ID | Item | Detail | Epic ID | Story ID | Jira key | Constraints | Mismatch / note | Flag OFF behaviour |
|---|---|---|---|---|---|---|---|---|
| M1-FLG-01-01 | sell.crm.01_01 | Leads & pipeline API | EP-MVP1-01 | ST-MVP1-01-01 | [SCRUM-91](https://rozvaa.atlassian.net/browse/SCRUM-91) (epic [SCRUM-70](https://rozvaa.atlassian.net/browse/SCRUM-70)) | C9 | key format <module>.<area>.<NN_NN> differs from MVP0 proposed keys — M1-MM-09 | OFF: hidden / 404 FEATURE_DISABLED; data readable, no writes lost (story) |
| M1-FLG-01-02 | sell.crm.01_02 | Follow-ups & customer 360 API | EP-MVP1-01 | ST-MVP1-01-02 | [SCRUM-92](https://rozvaa.atlassian.net/browse/SCRUM-92) (epic [SCRUM-70](https://rozvaa.atlassian.net/browse/SCRUM-70)) | C9 | key format <module>.<area>.<NN_NN> differs from MVP0 proposed keys — M1-MM-09 | OFF: hidden / 404 FEATURE_DISABLED; data readable, no writes lost (story) |
| M1-FLG-01-03 | sell.crm.01_03 | Quotation API | EP-MVP1-01 | ST-MVP1-01-03 | [SCRUM-95](https://rozvaa.atlassian.net/browse/SCRUM-95) (epic [SCRUM-70](https://rozvaa.atlassian.net/browse/SCRUM-70)) | C9 | key format <module>.<area>.<NN_NN> differs from MVP0 proposed keys — M1-MM-09 | OFF: hidden / 404 FEATURE_DISABLED; data readable, no writes lost (story) |
| M1-FLG-02-01 | sell.orders.02_01 | Sales order API with commitment & credit check | EP-MVP1-02 | ST-MVP1-02-01 | [SCRUM-96](https://rozvaa.atlassian.net/browse/SCRUM-96) (epic [SCRUM-71](https://rozvaa.atlassian.net/browse/SCRUM-71)) | C9 | key format <module>.<area>.<NN_NN> differs from MVP0 proposed keys — M1-MM-09 | OFF: hidden / 404 FEATURE_DISABLED; data readable, no writes lost (story) |
| M1-FLG-02-02 | sell.orders.02_02 | Material requirement & purchase impact API | EP-MVP1-02 | ST-MVP1-02-02 | [SCRUM-107](https://rozvaa.atlassian.net/browse/SCRUM-107) (epic [SCRUM-71](https://rozvaa.atlassian.net/browse/SCRUM-71)) | C9 | key format <module>.<area>.<NN_NN> differs from MVP0 proposed keys — M1-MM-09 | OFF: hidden / 404 FEATURE_DISABLED; data readable, no writes lost (story) |
| M1-FLG-01-04 | sell.crm.01_04 | Leads board, follow-ups and quotation screens | EP-MVP1-01 | ST-MVP1-01-04 | [SCRUM-114](https://rozvaa.atlassian.net/browse/SCRUM-114) (epic [SCRUM-70](https://rozvaa.atlassian.net/browse/SCRUM-70)) | C9 | key format <module>.<area>.<NN_NN> differs from MVP0 proposed keys — M1-MM-09 | OFF: hidden / 404 FEATURE_DISABLED; data readable, no writes lost (story) |
| M1-FLG-02-03 | sell.orders.02_03 | Orders list, detail, credit-hold and order documents | EP-MVP1-02 | ST-MVP1-02-03 | [SCRUM-125](https://rozvaa.atlassian.net/browse/SCRUM-125) (epic [SCRUM-71](https://rozvaa.atlassian.net/browse/SCRUM-71)) | C9 | key format <module>.<area>.<NN_NN> differs from MVP0 proposed keys — M1-MM-09 | OFF: hidden / 404 FEATURE_DISABLED; data readable, no writes lost (story) |

### MVP1 — Observability (DRAFT)

| Row ID | Item | Detail | Epic ID | Story ID | Jira key | Constraints | Mismatch / note |
|---|---|---|---|---|---|---|---|
| M1-OBS-SALES | business metrics: bos_leads_created_total{source}, bos_followups_overdue{owner}, bos_quotes_value_inr_total, bos_orders_on_credit_hold | RED + logs + spans + SLO + alerts + Grafana panel per story (values above) | EP-MVP1-01, EP-MVP1-02 | ST-MVP1-01-01, ST-MVP1-01-02, ST-MVP1-01-03, ST-MVP1-02-01, ST-MVP1-02-02, ST-MVP1-01-04, ST-MVP1-02-03 | [SCRUM-91](https://rozvaa.atlassian.net/browse/SCRUM-91), [SCRUM-92](https://rozvaa.atlassian.net/browse/SCRUM-92), [SCRUM-95](https://rozvaa.atlassian.net/browse/SCRUM-95), [SCRUM-96](https://rozvaa.atlassian.net/browse/SCRUM-96), [SCRUM-107](https://rozvaa.atlassian.net/browse/SCRUM-107), [SCRUM-114](https://rozvaa.atlassian.net/browse/SCRUM-114), [SCRUM-125](https://rozvaa.atlassian.net/browse/SCRUM-125) | C12 | story |

### MVP1 — UI → API (DRAFT)

| Row ID | Item | Detail | Epic ID | Story ID | Jira key | Constraints | Mismatch / note |
|---|---|---|---|---|---|---|---|
| M1-UI-01-04 | Leads board, follow-ups and quotation screens | Sell MFE in shell; optimistic stage drag with rollback. | EP-MVP1-01 | ST-MVP1-01-04 | [SCRUM-114](https://rozvaa.atlassian.net/browse/SCRUM-114) (epic [SCRUM-70](https://rozvaa.atlassian.net/browse/SCRUM-70)) | C9 | blocked by [SCRUM-91](https://rozvaa.atlassian.net/browse/SCRUM-91), [SCRUM-92](https://rozvaa.atlassian.net/browse/SCRUM-92), [SCRUM-95](https://rozvaa.atlassian.net/browse/SCRUM-95) |
| M1-UI-02-03 | Orders list, detail, credit-hold and order documents | Sell MFE. | EP-MVP1-02 | ST-MVP1-02-03 | [SCRUM-125](https://rozvaa.atlassian.net/browse/SCRUM-125) (epic [SCRUM-71](https://rozvaa.atlassian.net/browse/SCRUM-71)) | C9 |  |
