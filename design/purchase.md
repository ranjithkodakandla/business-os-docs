# purchase — MVP0/MVP1 design (DRAFT)

> **DRAFT — not frozen.** Design only: no OpenAPI, nothing provisioned. Open decisions OD1–OD9 are **UNCONFIRMED**. Mismatches with the Architect Doc are flagged, not resolved.

Sources: [Sheet tab 'MVP0-1 Design'](https://docs.google.com/spreadsheets/d/18p-aJ0QeW6L3SJ-CNSrNCjwKWBpVdkQAfkzeKOg40Oc/edit#gid=119904232) · [BOS Architect's Doc](https://docs.google.com/document/d/1sKoivvz3JLkKPI1Jta0i6XWnWuW57-rSwWICmm1w9VM/edit) · Jira SCRUM (MVP0 + MVP1 epics SCRUM-70..89, stories SCRUM-90..143). Generated 31 Oct 2026 IST.

Conventions, error/idempotency rules, quality gates, open decisions, mismatches and gaps: see [README](README.md). Rows: MVP0 9, MVP1 18.

## MVP0

### MVP0 — Data model (DRAFT)

| Row ID | Item | Detail | Epic ID | Story ID | Jira key | Constraints |
|---|---|---|---|---|---|---|
| TBL-PU-01 | suppliers, supplier_contacts | tenant_id, id, legal_name, gstin, gstin_status, state_code, msme_udyam; contacts: name, mobile, opted_in | EP-MVP0-12 | ST-MVP0-12-01 | [SCRUM-41](https://rozvaa.atlassian.net/browse/SCRUM-41) | C1 |
| TBL-PU-02 | rfqs, rfq_lines, quotes, counter_offers | tenant_id; rfq: due, status; lines: item_id, qty, uom; quote price_paise | EP-MVP0-12 | ST-MVP0-12-03 | [SCRUM-65](https://rozvaa.atlassian.net/browse/SCRUM-65) | C1, C15 |
| TBL-PU-03 | purchase_orders (incl. DRAFT), po_lines | tenant_id; status DRAFT\|PENDING_APPROVAL\|APPROVED\|SENT; amounts *_paise | EP-MVP0-12 | ST-MVP0-12-02 | [SCRUM-51](https://rozvaa.atlassian.net/browse/SCRUM-51) | C1, C15 |

### MVP0 — API contracts (DRAFT)

| Row ID | Item | Detail | Role / permission | Idempotency-Key | Emits / consumes | Epic ID | Story ID | Jira key | Constraints | Mismatch / note | p95 latency target | Payload limit |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| API-PU-01 | POST /v1/suppliers | {legal_name, gstin, state_code, msme_udyam, contacts[]} → {id, gstin_status}. GSTIN_* 422, SUPPLIER_DUPLICATE 409 | PURCHASE/ADMIN | Yes | purchase.supplier.created.v1 | EP-MVP0-12 | ST-MVP0-12-01 | [SCRUM-41](https://rozvaa.atlassian.net/browse/SCRUM-41) | C5 |  | <300 ms @50 rps (story DoD) | 1 MB (proposed) |
| API-PU-02 | POST /v1/rfqs; POST /v1/rfqs/{id}/counter | {items[], supplier_ids, due} → {id, status SENT} | PURCHASE | Yes | purchase.rfq.sent.v1 → notifications | EP-MVP0-12 | ST-MVP0-12-03 | [SCRUM-65](https://rozvaa.atlassian.net/browse/SCRUM-65) | C6, C13 | Counter request body not specified in story (GAP-S6). | <300 ms @50 rps (story DoD) | 1 MB (proposed) |

### MVP0 — Events (DRAFT)

| Row ID | Item | Detail | Emits / consumes | Epic ID | Story ID | Jira key | Constraints |
|---|---|---|---|---|---|---|---|
| EVT-09 | purchase.rfq.sent.v1 | consumers: notifications, audit; data: rfq_id, supplier_ids, due | producer: purchase | EP-MVP0-12 | ST-MVP0-12-03 | [SCRUM-65](https://rozvaa.atlassian.net/browse/SCRUM-65) | C4, C6, C12 |
| EVT-10 | purchase.po.drafted.v1 | consumers: ai-agents (approval if above limit), audit; data: po_id, total_paise | producer: purchase | EP-MVP0-12 | ST-MVP0-12-02 | [SCRUM-51](https://rozvaa.atlassian.net/browse/SCRUM-51) | C4, C6, C12 |

### MVP0 — Resilience (DRAFT)

| Row ID | Item | Detail | Epic ID | Story ID | Jira key | Constraints | Mismatch / note |
|---|---|---|---|---|---|---|---|
| RES-PU-01 | → notifications (RFQ send) | async via event; fallback: RFQ stays SENT_PENDING | EP-MVP0-12 | ST-MVP0-12-03 | [SCRUM-65](https://rozvaa.atlassian.net/browse/SCRUM-65) | C6, C12 | proposed |

### MVP0 — Observability (DRAFT)

| Row ID | Item | Detail | Epic ID | Story ID | Jira key | Constraints | Mismatch / note |
|---|---|---|---|---|---|---|---|
| OBS-PURCHA | RED + business metrics | RED per endpoint; business: rfqs_sent, quotes_received, po_drafts; JSON logs PII-masked (TST-06) with trace_id, hashed tenant_id; OTel spans incl. DB/Kafka/outbound; SLO: availability 99.9% (proposed unless story); alerts: burn-rate 2%/1h & 5%/6h, DLQ>0, breaker open; Grafana panel "purchase" | EP-MVP0-06 | ST-MVP0-06-01, ST-MVP0-06-02 | [SCRUM-22](https://rozvaa.atlassian.net/browse/SCRUM-22), [SCRUM-28](https://rozvaa.atlassian.net/browse/SCRUM-28) | C12 | proposed |

## MVP1

### MVP1 — Data model (DRAFT)

| Row ID | Item | Detail | Epic ID | Story ID | Jira key | Constraints |
|---|---|---|---|---|---|---|
| M1-TBL-PURC-01 | prs, rfq_quotes, pos | tenant_id + RLS policy; story | EP-MVP1-04 | ST-MVP1-04-01 | [SCRUM-98](https://rozvaa.atlassian.net/browse/SCRUM-98) (epic [SCRUM-73](https://rozvaa.atlassian.net/browse/SCRUM-73)) | C1, C3, C15 |
| M1-TBL-PURC-02 | bill_scan_jobs, draft_bills | tenant_id + RLS policy; proposed | EP-MVP1-04 | ST-MVP1-04-02 | [SCRUM-127](https://rozvaa.atlassian.net/browse/SCRUM-127) (epic [SCRUM-73](https://rozvaa.atlassian.net/browse/SCRUM-73)) | C1, C3, C15 |
| M1-TBL-PURC-03 | match_runs, match_exceptions (per-tenant tolerance) | tenant_id + RLS policy; proposed | EP-MVP1-18 | ST-MVP1-18-01 | [SCRUM-113](https://rozvaa.atlassian.net/browse/SCRUM-113) (epic [SCRUM-87](https://rozvaa.atlassian.net/browse/SCRUM-87)) | C1, C3, C15 |

### MVP1 — API contracts (DRAFT)

| Row ID | Item | Detail | Role / permission | Idempotency-Key | Emits / consumes | Epic ID | Story ID | Jira key | Constraints | Mismatch / note | p95 latency target | Payload limit |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| M1-API-04-01 | POST /v1/prs ; POST /v1/prs/{id}/compare ; POST /v1/pos | Req: {"pr_id":"pr_3","supplier_id":"sup_2","lines":[{"item_id":"it_1","qty":300,"rate":198}],"terms_days":30} Resp: {"id":"po_5","no":"PO/26-27/0120","status":"PENDING_APPROVAL"} Errors: MSE_TERMS_EXCEED_45 422, PR_ALREADY_ORDERED 409 | PURCHASE | Yes (POST) | outbox events per M1-3 | EP-MVP1-04 | ST-MVP1-04-01 | [SCRUM-98](https://rozvaa.atlassian.net/browse/SCRUM-98) (epic [SCRUM-73](https://rozvaa.atlassian.net/browse/SCRUM-73)) | C2, C5, C6, C12 | blocked by [SCRUM-65](https://rozvaa.atlassian.net/browse/SCRUM-65), [SCRUM-37](https://rozvaa.atlassian.net/browse/SCRUM-37) | <500 ms (story ASSUMPTION; MVP0 DoD <300 ms — M1-MM-02) | 1 MB JSON (story) |
| M1-API-04-02 | POST /v1/bills/scan (multipart) ; GET /v1/bills/scan/{job} | Req: file=bill.jpg Resp: {"job_id":"ocr_1","status":"DONE","draft_bill_id":"pb_9","fields":{"supplier_gstin":{"value":"27AAACR5055K1Z7","conf":0.97}}} Errors: FILE_TOO_LARGE 413 (>10 MB ASSUMPTION), UNSUPPORTED_TYPE 415, OCR_FAILED 422 | ACCOUNTANT/PURCHASE | Yes (POST) | outbox events per M1-3 | EP-MVP1-04 | ST-MVP1-04-02 | [SCRUM-127](https://rozvaa.atlassian.net/browse/SCRUM-127) (epic [SCRUM-73](https://rozvaa.atlassian.net/browse/SCRUM-73)) | C2, C5, C6, C12 | blocked by [SCRUM-98](https://rozvaa.atlassian.net/browse/SCRUM-98) | <500 ms (story ASSUMPTION; MVP0 DoD <300 ms — M1-MM-02) | 10 MB file (story ASSUMPTION) |
| M1-API-18-01 | POST /v1/match/run?doc= ; POST /v1/match/{id}/resolve | Req: {"action":"REQUEST_DEBIT_NOTE","note":"Rate as per PO"} Resp: {"id":"mt_1","status":"RESOLVED"} Errors: RESOLUTION_NEEDS_APPROVAL 202 | ACCOUNTANT | Yes (POST) | outbox events per M1-3 | EP-MVP1-18 | ST-MVP1-18-01 | [SCRUM-113](https://rozvaa.atlassian.net/browse/SCRUM-113) (epic [SCRUM-87](https://rozvaa.atlassian.net/browse/SCRUM-87)) | C2, C5, C6, C12 | blocked by [SCRUM-98](https://rozvaa.atlassian.net/browse/SCRUM-98), [SCRUM-96](https://rozvaa.atlassian.net/browse/SCRUM-96), [SCRUM-93](https://rozvaa.atlassian.net/browse/SCRUM-93) | <500 ms (story ASSUMPTION; MVP0 DoD <300 ms — M1-MM-02) | 1 MB JSON (story) |

### MVP1 — Events (DRAFT)

| Row ID | Item | Detail | Emits / consumes | Epic ID | Story ID | Jira key | Constraints |
|---|---|---|---|---|---|---|---|
| M1-EVT-05 | po.created.v1 / po.approved.v1 | story POCreated/POApproved; consumers: inventory, notifications, audit | producer: purchase | EP-MVP1-04 | ST-MVP1-04-01 | [SCRUM-98](https://rozvaa.atlassian.net/browse/SCRUM-98) (epic [SCRUM-73](https://rozvaa.atlassian.net/browse/SCRUM-73)) | C4, C6, C12 |
| M1-EVT-06 | po.date_changed.v1 | story PODateChanged; consumers: sales-crm (MRP) | producer: purchase | EP-MVP1-02 | ST-MVP1-02-02 | [SCRUM-107](https://rozvaa.atlassian.net/browse/SCRUM-107) (epic [SCRUM-71](https://rozvaa.atlassian.net/browse/SCRUM-71)) | C4, C6, C12 |

### MVP1 — Feature flags (with OFF behaviour) (DRAFT)

| Row ID | Item | Detail | Epic ID | Story ID | Jira key | Constraints | Mismatch / note | Flag OFF behaviour |
|---|---|---|---|---|---|---|---|---|
| M1-FLG-04-01 | buy.procurement.04_01 | Purchase request, quote comparison & PO API | EP-MVP1-04 | ST-MVP1-04-01 | [SCRUM-98](https://rozvaa.atlassian.net/browse/SCRUM-98) (epic [SCRUM-73](https://rozvaa.atlassian.net/browse/SCRUM-73)) | C9 | key format <module>.<area>.<NN_NN> differs from MVP0 proposed keys — M1-MM-09 | OFF: hidden / 404 FEATURE_DISABLED; data readable, no writes lost (story) |
| M1-FLG-04-02 | buy.procurement.04_02 | Bill & receipt autoscan (OCR) to draft purchase bill | EP-MVP1-04 | ST-MVP1-04-02 | [SCRUM-127](https://rozvaa.atlassian.net/browse/SCRUM-127) (epic [SCRUM-73](https://rozvaa.atlassian.net/browse/SCRUM-73)) | C9 | key format <module>.<area>.<NN_NN> differs from MVP0 proposed keys — M1-MM-09 | OFF: hidden / 404 FEATURE_DISABLED; data readable, no writes lost (story) |
| M1-FLG-18-01 | money.match.18_01 | Three-way match engine (PO-GRN-bill, SO-OC-invoice) with AI reason & resolve | EP-MVP1-18 | ST-MVP1-18-01 | [SCRUM-113](https://rozvaa.atlassian.net/browse/SCRUM-113) (epic [SCRUM-87](https://rozvaa.atlassian.net/browse/SCRUM-87)) | C9 | key format <module>.<area>.<NN_NN> differs from MVP0 proposed keys — M1-MM-09 | OFF: hidden / 404 FEATURE_DISABLED; data readable, no writes lost (story) |
| M1-FLG-18-02 | money.match.18_02 | Match workbench and mobile swipe approvals (PWA) | EP-MVP1-18 | ST-MVP1-18-02 | [SCRUM-126](https://rozvaa.atlassian.net/browse/SCRUM-126) (epic [SCRUM-87](https://rozvaa.atlassian.net/browse/SCRUM-87)) | C9 | key format <module>.<area>.<NN_NN> differs from MVP0 proposed keys — M1-MM-09 | OFF: hidden / 404 FEATURE_DISABLED; data readable, no writes lost (story) |
| M1-FLG-04-03 | buy.procurement.04_03 | Purchase requests, comparison matrix, PO and autoscan screens | EP-MVP1-04 | ST-MVP1-04-03 | [SCRUM-139](https://rozvaa.atlassian.net/browse/SCRUM-139) (epic [SCRUM-73](https://rozvaa.atlassian.net/browse/SCRUM-73)) | C9 | key format <module>.<area>.<NN_NN> differs from MVP0 proposed keys — M1-MM-09 | OFF: hidden / 404 FEATURE_DISABLED; data readable, no writes lost (story) |

### MVP1 — Resilience (DRAFT)

| Row ID | Item | Detail | Epic ID | Story ID | Jira key | Constraints | Mismatch / note |
|---|---|---|---|---|---|---|---|
| M1-RES-04-02 | cb.llm_vision_api | timeout 10 s external; breaker 50%/20 calls, half-open 30 s; bulkhead 10/tenant; fallback: degrade to manual / template text, never block the core flow | EP-MVP1-04 | ST-MVP1-04-02 | [SCRUM-127](https://rozvaa.atlassian.net/browse/SCRUM-127) (epic [SCRUM-73](https://rozvaa.atlassian.net/browse/SCRUM-73)) | C6, C12 | story |
| M1-RES-18-01 | cb.llm_api | timeout 10 s external; breaker 50%/20 calls, half-open 30 s; bulkhead 10/tenant; fallback: degrade to manual / template text, never block the core flow | EP-MVP1-18 | ST-MVP1-18-01 | [SCRUM-113](https://rozvaa.atlassian.net/browse/SCRUM-113) (epic [SCRUM-87](https://rozvaa.atlassian.net/browse/SCRUM-87)) | C6, C12 | story |

### MVP1 — Observability (DRAFT)

| Row ID | Item | Detail | Epic ID | Story ID | Jira key | Constraints | Mismatch / note |
|---|---|---|---|---|---|---|---|
| M1-OBS-PURCHA | business metrics: bos_po_value_inr_total, bos_ocr_field_accuracy, bos_match_mismatch_total{type} | RED + logs + spans + SLO + alerts + Grafana panel per story (values above) | EP-MVP1-04, EP-MVP1-18 | ST-MVP1-04-01, ST-MVP1-04-02, ST-MVP1-18-01, ST-MVP1-18-02, ST-MVP1-04-03 | [SCRUM-98](https://rozvaa.atlassian.net/browse/SCRUM-98), [SCRUM-127](https://rozvaa.atlassian.net/browse/SCRUM-127), [SCRUM-113](https://rozvaa.atlassian.net/browse/SCRUM-113), [SCRUM-126](https://rozvaa.atlassian.net/browse/SCRUM-126), [SCRUM-139](https://rozvaa.atlassian.net/browse/SCRUM-139) | C12 | story |

### MVP1 — UI → API (DRAFT)

| Row ID | Item | Detail | Epic ID | Story ID | Jira key | Constraints | Mismatch / note |
|---|---|---|---|---|---|---|---|
| M1-UI-18-02 | Match workbench and mobile swipe approvals (PWA) | Service worker; push. | EP-MVP1-18 | ST-MVP1-18-02 | [SCRUM-126](https://rozvaa.atlassian.net/browse/SCRUM-126) (epic [SCRUM-87](https://rozvaa.atlassian.net/browse/SCRUM-87)) | C9 | blocked by [SCRUM-113](https://rozvaa.atlassian.net/browse/SCRUM-113), [SCRUM-37](https://rozvaa.atlassian.net/browse/SCRUM-37) |
| M1-UI-04-03 | Purchase requests, comparison matrix, PO and autoscan screens | Buy MFE. | EP-MVP1-04 | ST-MVP1-04-03 | [SCRUM-139](https://rozvaa.atlassian.net/browse/SCRUM-139) (epic [SCRUM-73](https://rozvaa.atlassian.net/browse/SCRUM-73)) | C9 | blocked by [SCRUM-98](https://rozvaa.atlassian.net/browse/SCRUM-98), [SCRUM-127](https://rozvaa.atlassian.net/browse/SCRUM-127) |
