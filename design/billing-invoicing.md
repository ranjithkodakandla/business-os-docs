# billing-invoicing — MVP0/MVP1 design (DRAFT)

> **DRAFT — not frozen.** Design only: no OpenAPI, nothing provisioned. Open decisions OD1–OD9 are **UNCONFIRMED**. Mismatches with the Architect Doc are flagged, not resolved.

Sources: [Sheet tab 'MVP0-1 Design'](https://docs.google.com/spreadsheets/d/18p-aJ0QeW6L3SJ-CNSrNCjwKWBpVdkQAfkzeKOg40Oc/edit#gid=119904232) · [BOS Architect's Doc](https://docs.google.com/document/d/1sKoivvz3JLkKPI1Jta0i6XWnWuW57-rSwWICmm1w9VM/edit) · Jira SCRUM (MVP0 + MVP1 epics SCRUM-70..89, stories SCRUM-90..143). Generated 31 Oct 2026 IST.

Conventions, error/idempotency rules, quality gates, open decisions, mismatches and gaps: see [README](README.md). Rows: MVP0 7, MVP1 21.

## MVP0

### MVP0 — Data model (DRAFT)

| Row ID | Item | Detail | Epic ID | Story ID | Jira key | Constraints | Mismatch / note |
|---|---|---|---|---|---|---|---|
| TBL-BI-01 | document_series | tenant_id, id, doc_type, unit_id, pattern, fy, start_at, next_seq, reset, locked. Rendered number ≤16 chars. | EP-MVP0-01 | ST-MVP0-01-05 | [SCRUM-29](https://rozvaa.atlassian.net/browse/SCRUM-29) | C1 | Owner not named in Doc; placed in billing (MM-13). |
| TBL-BI-02 | invoices, invoice_lines, tax_breakups | amounts *_paise bigint; tax_breakups: cgst_paise, sgst_paise, igst_paise, cess_paise per line | NO STORY | NO STORY | — | C1, C15 | No MVP0 story creates invoices, but 13-01/13-03 reference invoice_id. |
| TBL-BI-03 | compliance_calendar (gst_returns) | tenant_id, form, period, due, status | EP-MVP0-10 | ST-MVP0-10-02 | [SCRUM-49](https://rozvaa.atlassian.net/browse/SCRUM-49) | C1 |  |

### MVP0 — API contracts (DRAFT)

| Row ID | Item | Detail | Role / permission | Idempotency-Key | Emits / consumes | Epic ID | Story ID | Jira key | Constraints | Mismatch / note | p95 latency target | Payload limit |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| API-BI-01 | POST /v1/document-series | {doc_type, unit_id, pattern, fy, start_at, reset} → {id, preview[], max_len, locked}. SERIES_TOO_LONG 422, SERIES_LOCKED 409, SERIES_DUPLICATE 409 | OWNER/ADMIN/ACCOUNTANT | Yes | — | EP-MVP0-01 | ST-MVP0-01-05 | [SCRUM-29](https://rozvaa.atlassian.net/browse/SCRUM-29) | C5 |  | <300 ms @50 rps (story DoD) | 1 MB (proposed) |
| API-BI-02 | GET /v1/compliance-calendar?month= | → {items[{form, due, status}]} | ACCOUNTANT/OWNER | n/a (GET) | — | EP-MVP0-10 | ST-MVP0-10-02 | [SCRUM-49](https://rozvaa.atlassian.net/browse/SCRUM-49) | C5 | Doc lists /v1/gst-returns/{period}; calendar path not in Doc (minor). | <300 ms @50 rps (story DoD) | 1 MB (proposed) |

### MVP0 — Events (DRAFT)

| Row ID | Item | Detail | Emits / consumes | Epic ID | Story ID | Jira key | Constraints | Mismatch / note |
|---|---|---|---|---|---|---|---|---|
| EVT-17 | invoice.issued.v1 | consumers: collections-payments (receivables), documents, audit; data: invoice_id, party_id, total_paise, tax_breakup | producer: billing-invoicing | NO STORY | NO STORY | — | C4, C6, C12 | Needed by MVP0 receivables but no MVP0 story produces it. |

### MVP0 — Observability (DRAFT)

| Row ID | Item | Detail | Epic ID | Story ID | Jira key | Constraints | Mismatch / note |
|---|---|---|---|---|---|---|---|
| OBS-BILLIN | RED + business metrics | RED per endpoint; business: invoices_issued, series_gaps (must be 0); JSON logs PII-masked (TST-06) with trace_id, hashed tenant_id; OTel spans incl. DB/Kafka/outbound; SLO: issue success ≥99.9% (proposed unless story); alerts: burn-rate 2%/1h & 5%/6h, DLQ>0, breaker open; Grafana panel "billing-invoicing" | EP-MVP0-06 | ST-MVP0-06-01, ST-MVP0-06-02 | [SCRUM-22](https://rozvaa.atlassian.net/browse/SCRUM-22), [SCRUM-28](https://rozvaa.atlassian.net/browse/SCRUM-28) | C12 | proposed |

## MVP1

### MVP1 — Data model (DRAFT)

| Row ID | Item | Detail | Epic ID | Story ID | Jira key | Constraints |
|---|---|---|---|---|---|---|
| M1-TBL-BILL-01 | einvoice_irn (irn, ack_no, signed_qr) | tenant_id + RLS policy; proposed; GSP creds in Secret Manager | EP-MVP1-03 | ST-MVP1-03-01 | [SCRUM-97](https://rozvaa.atlassian.net/browse/SCRUM-97) (epic [SCRUM-72](https://rozvaa.atlassian.net/browse/SCRUM-72)) | C1, C3, C15 |
| M1-TBL-BILL-02 | ewaybills | tenant_id + RLS policy; proposed | EP-MVP1-03 | ST-MVP1-03-02 | [SCRUM-108](https://rozvaa.atlassian.net/browse/SCRUM-108) (epic [SCRUM-72](https://rozvaa.atlassian.net/browse/SCRUM-72)) | C1, C3, C15 |
| M1-TBL-BILL-03 | gst_return_snapshots (per period, locked) | tenant_id + RLS policy; story: snapshot + period lock | EP-MVP1-03 | ST-MVP1-03-03 | [SCRUM-133](https://rozvaa.atlassian.net/browse/SCRUM-133) (epic [SCRUM-72](https://rozvaa.atlassian.net/browse/SCRUM-72)) | C1, C3, C15 |
| M1-TBL-BILL-04 | report_cache (15 min) on read replica | tenant_id + RLS policy; story: read replica; owner unassigned (M1-MM-05) | EP-MVP1-09 | ST-MVP1-09-01 | [SCRUM-109](https://rozvaa.atlassian.net/browse/SCRUM-109) (epic [SCRUM-78](https://rozvaa.atlassian.net/browse/SCRUM-78)) | C1, C3, C15 |

### MVP1 — API contracts (DRAFT)

| Row ID | Item | Detail | Role / permission | Idempotency-Key | Emits / consumes | Epic ID | Story ID | Jira key | Constraints | Mismatch / note | p95 latency target | Payload limit |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| M1-API-03-01 | POST /v1/invoices/{id}/irn ; POST /v1/invoices/{id}/irn/cancel | Req: {} (header Idempotency-Key) Resp: {"status":"IRN_GENERATED","irn":"a1b2...64hex","ack_no":"112610012345678","ack_dt":"2026-10-30T10:05:00+05:30","signed_qr":"eyJ..."} Errors: IRP_WINDOW_EXPIRED 422, EINV_NOT_APPLICABLE 422, GSP_VALIDATION 422 (maps IRP code), GSP_UNAVAILABLE 503 (queued retry), DUPLICATE_IRN 409 (fetch existing) | ACCOUNTANT | Yes (POST) | outbox events per M1-3 | EP-MVP1-03 | ST-MVP1-03-01 | [SCRUM-97](https://rozvaa.atlassian.net/browse/SCRUM-97) (epic [SCRUM-72](https://rozvaa.atlassian.net/browse/SCRUM-72)) | C2, C5, C6, C12 | blocked by [SCRUM-54](https://rozvaa.atlassian.net/browse/SCRUM-54), [SCRUM-96](https://rozvaa.atlassian.net/browse/SCRUM-96) | <500 ms (story ASSUMPTION; MVP0 DoD <300 ms — M1-MM-02) | 1 MB JSON (story) |
| M1-API-03-02 | POST /v1/invoices/{id}/ewb ; PATCH /v1/ewb/{no}/vehicle ; POST /v1/ewb/{no}/cancel | Req: {"transport_mode":"ROAD","vehicle_no":"MH12AB1234","distance_km":148,"transporter_gstin":null} Resp: {"ewb_no":"331009876543","valid_upto":"2026-10-31T23:59:00+05:30"} Errors: EWB_NOT_REQUIRED 422, VEHICLE_FORMAT 422, GSP_UNAVAILABLE 503 | ACCOUNTANT/STORE | Yes (POST) | outbox events per M1-3 | EP-MVP1-03 | ST-MVP1-03-02 | [SCRUM-108](https://rozvaa.atlassian.net/browse/SCRUM-108) (epic [SCRUM-72](https://rozvaa.atlassian.net/browse/SCRUM-72)) | C2, C5, C6, C12 |  | <500 ms (story ASSUMPTION; MVP0 DoD <300 ms — M1-MM-02) | 1 MB JSON (story) |
| M1-API-03-03 | POST /v1/gst/returns/{type}/{period}/prepare ; .../approve ; .../export | Req: {"filing_freq":"MONTHLY"} Resp: {"status":"PREPARED","sections":{"b2b":{"count":42,"taxable":2450000.00}},"warnings":[{"code":"HSN_MISSING","doc":"INV/26-27/0101"}]} Errors: PERIOD_LOCKED 409, UNPOSTED_DOCS 422 | ACCOUNTANT (prepare), OWNER/CA (approve) | Yes (POST) | outbox events per M1-3 | EP-MVP1-03 | ST-MVP1-03-03 | [SCRUM-133](https://rozvaa.atlassian.net/browse/SCRUM-133) (epic [SCRUM-72](https://rozvaa.atlassian.net/browse/SCRUM-72)) | C2, C5, C6, C12 |  | <500 ms (story ASSUMPTION; MVP0 DoD <300 ms — M1-MM-02) | 1 MB JSON (story) |
| M1-API-09-01 | GET /v1/reports/{key}?from=&to=&group_by= | Req: - Resp: {"key":"dso","value":37.5,"unit":"days","sources":["/v1/ageing/receivables"]} Errors: REPORT_UNKNOWN 404 | OWNER/ACCOUNTANT | n/a | outbox events per M1-3 | EP-MVP1-09 | ST-MVP1-09-01 | [SCRUM-109](https://rozvaa.atlassian.net/browse/SCRUM-109) (epic [SCRUM-78](https://rozvaa.atlassian.net/browse/SCRUM-78)) | C2, C5, C6, C12 | blocked by [SCRUM-94](https://rozvaa.atlassian.net/browse/SCRUM-94), [SCRUM-93](https://rozvaa.atlassian.net/browse/SCRUM-93) | <500 ms (story ASSUMPTION; MVP0 DoD <300 ms — M1-MM-02) | 1 MB JSON (story) |

### MVP1 — Events (DRAFT)

| Row ID | Item | Detail | Emits / consumes | Epic ID | Story ID | Jira key | Constraints |
|---|---|---|---|---|---|---|---|
| M1-EVT-07 | einvoice.irn_generated.v1 | proposed (async via outbox); consumers: documents (QR on print), notifications | producer: billing-invoicing | EP-MVP1-03 | ST-MVP1-03-01 | [SCRUM-97](https://rozvaa.atlassian.net/browse/SCRUM-97) (epic [SCRUM-72](https://rozvaa.atlassian.net/browse/SCRUM-72)) | C4, C6, C12 |

### MVP1 — Feature flags (with OFF behaviour) (DRAFT)

| Row ID | Item | Detail | Epic ID | Story ID | Jira key | Constraints | Mismatch / note | Flag OFF behaviour |
|---|---|---|---|---|---|---|---|---|
| M1-FLG-03-01 | money.gst.03_01 | e-Invoice IRN generation & cancellation API | EP-MVP1-03 | ST-MVP1-03-01 | [SCRUM-97](https://rozvaa.atlassian.net/browse/SCRUM-97) (epic [SCRUM-72](https://rozvaa.atlassian.net/browse/SCRUM-72)) | C9 | key format <module>.<area>.<NN_NN> differs from MVP0 proposed keys — M1-MM-09 | OFF: hidden / 404 FEATURE_DISABLED; data readable, no writes lost (story) |
| M1-FLG-03-02 | money.gst.03_02 | e-Way bill API (Part A/B, update, cancel) | EP-MVP1-03 | ST-MVP1-03-02 | [SCRUM-108](https://rozvaa.atlassian.net/browse/SCRUM-108) (epic [SCRUM-72](https://rozvaa.atlassian.net/browse/SCRUM-72)) | C9 | key format <module>.<area>.<NN_NN> differs from MVP0 proposed keys — M1-MM-09 | OFF: hidden / 404 FEATURE_DISABLED; data readable, no writes lost (story) |
| M1-FLG-03-03 | money.gst.03_03 | GSTR-1 & GSTR-3B preparation API with maker-checker | EP-MVP1-03 | ST-MVP1-03-03 | [SCRUM-133](https://rozvaa.atlassian.net/browse/SCRUM-133) (epic [SCRUM-72](https://rozvaa.atlassian.net/browse/SCRUM-72)) | C9 | key format <module>.<area>.<NN_NN> differs from MVP0 proposed keys — M1-MM-09 | OFF: hidden / 404 FEATURE_DISABLED; data readable, no writes lost (story) |
| M1-FLG-03-04 | money.gst.03_04 | Invoice compliance panel & GST returns screens | EP-MVP1-03 | ST-MVP1-03-04 | [SCRUM-138](https://rozvaa.atlassian.net/browse/SCRUM-138) (epic [SCRUM-72](https://rozvaa.atlassian.net/browse/SCRUM-72)) | C9 | key format <module>.<area>.<NN_NN> differs from MVP0 proposed keys — M1-MM-09 | OFF: hidden / 404 FEATURE_DISABLED; data readable, no writes lost (story) |
| M1-FLG-09-01 | money.reports.09_01 | Money reports API (billing vs collections, DSO, inventory value, margin/OTD/atte | EP-MVP1-09 | ST-MVP1-09-01 | [SCRUM-109](https://rozvaa.atlassian.net/browse/SCRUM-109) (epic [SCRUM-78](https://rozvaa.atlassian.net/browse/SCRUM-78)) | C9 | key format <module>.<area>.<NN_NN> differs from MVP0 proposed keys — M1-MM-09 | OFF: hidden / 404 FEATURE_DISABLED; data readable, no writes lost (story) |
| M1-FLG-09-03 | money.reports.09_03 | Reports hub with AI narrative | EP-MVP1-09 | ST-MVP1-09-03 | [SCRUM-136](https://rozvaa.atlassian.net/browse/SCRUM-136) (epic [SCRUM-78](https://rozvaa.atlassian.net/browse/SCRUM-78)) | C9 | key format <module>.<area>.<NN_NN> differs from MVP0 proposed keys — M1-MM-09 | OFF: hidden / 404 FEATURE_DISABLED; data readable, no writes lost (story) |

### MVP1 — Resilience (DRAFT)

| Row ID | Item | Detail | Epic ID | Story ID | Jira key | Constraints | Mismatch / note |
|---|---|---|---|---|---|---|---|
| M1-RES-03-01 | cb.gsp_e_invoice_api | timeout 10 s external; breaker 50%/20 calls, half-open 30 s; bulkhead 10/tenant; fallback: queue and retry, show PENDING | EP-MVP1-03 | ST-MVP1-03-01 | [SCRUM-97](https://rozvaa.atlassian.net/browse/SCRUM-97) (epic [SCRUM-72](https://rozvaa.atlassian.net/browse/SCRUM-72)) | C6, C12 | story |
| M1-RES-03-02 | cb.gsp_e_way_bill_api | timeout 10 s external; breaker 50%/20 calls, half-open 30 s; bulkhead 10/tenant; fallback: queue and retry, show PENDING | EP-MVP1-03 | ST-MVP1-03-02 | [SCRUM-108](https://rozvaa.atlassian.net/browse/SCRUM-108) (epic [SCRUM-72](https://rozvaa.atlassian.net/browse/SCRUM-72)) | C6, C12 | story |
| M1-RES-03-03 | cb.gsp_returns_api_optional | timeout 10 s external; breaker 50%/20 calls, half-open 30 s; bulkhead 10/tenant; fallback: queue and retry, show PENDING | EP-MVP1-03 | ST-MVP1-03-03 | [SCRUM-133](https://rozvaa.atlassian.net/browse/SCRUM-133) (epic [SCRUM-72](https://rozvaa.atlassian.net/browse/SCRUM-72)) | C6, C12 | story |

### MVP1 — Observability (DRAFT)

| Row ID | Item | Detail | Epic ID | Story ID | Jira key | Constraints | Mismatch / note |
|---|---|---|---|---|---|---|---|
| M1-OBS-BILLIN | business metrics: bos_irn_generated_total{result}, bos_ewb_generated_total{result} | RED + logs + spans + SLO + alerts + Grafana panel per story (values above) | EP-MVP1-03, EP-MVP1-09 | ST-MVP1-03-01, ST-MVP1-03-02, ST-MVP1-03-03, ST-MVP1-03-04, ST-MVP1-09-01, ST-MVP1-09-03 | [SCRUM-97](https://rozvaa.atlassian.net/browse/SCRUM-97), [SCRUM-108](https://rozvaa.atlassian.net/browse/SCRUM-108), [SCRUM-133](https://rozvaa.atlassian.net/browse/SCRUM-133), [SCRUM-138](https://rozvaa.atlassian.net/browse/SCRUM-138), [SCRUM-109](https://rozvaa.atlassian.net/browse/SCRUM-109), [SCRUM-136](https://rozvaa.atlassian.net/browse/SCRUM-136) | C12 | story |

### MVP1 — UI → API (DRAFT)

| Row ID | Item | Detail | Epic ID | Story ID | Jira key | Constraints | Mismatch / note |
|---|---|---|---|---|---|---|---|
| M1-UI-03-04 | Invoice compliance panel & GST returns screens | Money MFE (M5). | EP-MVP1-03 | ST-MVP1-03-04 | [SCRUM-138](https://rozvaa.atlassian.net/browse/SCRUM-138) (epic [SCRUM-72](https://rozvaa.atlassian.net/browse/SCRUM-72)) | C9 | blocked by [SCRUM-97](https://rozvaa.atlassian.net/browse/SCRUM-97), [SCRUM-108](https://rozvaa.atlassian.net/browse/SCRUM-108), [SCRUM-133](https://rozvaa.atlassian.net/browse/SCRUM-133) |
| M1-UI-09-03 | Reports hub with AI narrative | Money MFE. | EP-MVP1-09 | ST-MVP1-09-03 | [SCRUM-136](https://rozvaa.atlassian.net/browse/SCRUM-136) (epic [SCRUM-78](https://rozvaa.atlassian.net/browse/SCRUM-78)) | C9 |  |
