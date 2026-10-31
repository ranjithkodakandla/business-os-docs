# inventory — MVP0/MVP1 design (DRAFT)

> **DRAFT — not frozen.** Design only: no OpenAPI, nothing provisioned. Open decisions OD1–OD9 are **UNCONFIRMED**. Mismatches with the Architect Doc are flagged, not resolved.

Sources: [Sheet tab 'MVP0-1 Design'](https://docs.google.com/spreadsheets/d/18p-aJ0QeW6L3SJ-CNSrNCjwKWBpVdkQAfkzeKOg40Oc/edit#gid=119904232) · [BOS Architect's Doc](https://docs.google.com/document/d/1sKoivvz3JLkKPI1Jta0i6XWnWuW57-rSwWICmm1w9VM/edit) · Jira SCRUM (MVP0 + MVP1 epics SCRUM-70..89, stories SCRUM-90..143). Generated 31 Oct 2026 IST.

Conventions, error/idempotency rules, quality gates, open decisions, mismatches and gaps: see [README](README.md). Rows: MVP0 5, MVP1 10.

## MVP0

### MVP0 — Data model (DRAFT)

| Row ID | Item | Detail | Epic ID | Story ID | Jira key | Constraints | Mismatch / note |
|---|---|---|---|---|---|---|---|
| TBL-IN-01 | items, uoms, stock_balances, boms, bom_lines | tenant_id on each; qty numeric; bom_lines(fg_item_id, rm_item_id, qty_per) | EP-MVP0-12 | ST-MVP0-12-02 | [SCRUM-51](https://rozvaa.atlassian.net/browse/SCRUM-51) | C1 | BOM not listed in Doc's inventory tables (MM-15). |

### MVP0 — API contracts (DRAFT)

| Row ID | Item | Detail | Role / permission | Idempotency-Key | Emits / consumes | Epic ID | Story ID | Jira key | Constraints | Mismatch / note | p95 latency target | Payload limit |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| API-IN-01 | POST /v1/bom-check | {item_id, qty} → {shortages[], po_draft_id} | PURCHASE/STORE | Yes | inventory.shortage.detected.v1 → purchase creates PO draft | EP-MVP0-12 | ST-MVP0-12-02 | [SCRUM-51](https://rozvaa.atlassian.net/browse/SCRUM-51) | C4, C6 | Sync po_draft_id needs cross-service call; async-first says event (MM-15). | <300 ms @50 rps (story DoD) | 1 MB (proposed) |

### MVP0 — Events (DRAFT)

| Row ID | Item | Detail | Emits / consumes | Epic ID | Story ID | Jira key | Constraints |
|---|---|---|---|---|---|---|---|
| EVT-08 | inventory.shortage.detected.v1 | consumers: purchase (create PO draft), ai-agents; data: item_id, qty, shortages[] | producer: inventory | EP-MVP0-12 | ST-MVP0-12-02 | [SCRUM-51](https://rozvaa.atlassian.net/browse/SCRUM-51) | C4, C6, C12 |

### MVP0 — Resilience (DRAFT)

| Row ID | Item | Detail | Epic ID | Story ID | Jira key | Constraints | Mismatch / note |
|---|---|---|---|---|---|---|---|
| RES-IN-01 | → purchase (PO draft, sync in story) | timeout 2 s; no retry w/o Idempotency-Key; breaker; fallback: return shortages without po_draft_id; draft created async (MM-15) | EP-MVP0-12 | ST-MVP0-12-02 | [SCRUM-51](https://rozvaa.atlassian.net/browse/SCRUM-51) | C6, C12 | proposed |

### MVP0 — Observability (DRAFT)

| Row ID | Item | Detail | Epic ID | Story ID | Jira key | Constraints | Mismatch / note |
|---|---|---|---|---|---|---|---|
| OBS-INVENT | RED + business metrics | RED per endpoint; business: bom_checks, shortages_detected; JSON logs PII-masked (TST-06) with trace_id, hashed tenant_id; OTel spans incl. DB/Kafka/outbound; SLO: availability 99.9% (proposed unless story); alerts: burn-rate 2%/1h & 5%/6h, DLQ>0, breaker open; Grafana panel "inventory" | EP-MVP0-06 | ST-MVP0-06-01, ST-MVP0-06-02 | [SCRUM-22](https://rozvaa.atlassian.net/browse/SCRUM-22), [SCRUM-28](https://rozvaa.atlassian.net/browse/SCRUM-28) | C12 | proposed |

## MVP1

### MVP1 — Data model (DRAFT)

| Row ID | Item | Detail | Epic ID | Story ID | Jira key | Constraints |
|---|---|---|---|---|---|---|
| M1-TBL-INVE-01 | stock_movements (append-only), stock_balances | tenant_id + RLS policy; balances updated in same tx (story) | EP-MVP1-05 | ST-MVP1-05-01 | [SCRUM-93](https://rozvaa.atlassian.net/browse/SCRUM-93) (epic [SCRUM-74](https://rozvaa.atlassian.net/browse/SCRUM-74)) | C1, C3, C15 |
| M1-TBL-INVE-02 | batches, reservations unique (order_line, batch), reorder_rules | tenant_id + RLS policy; story | EP-MVP1-05 | ST-MVP1-05-02 | [SCRUM-99](https://rozvaa.atlassian.net/browse/SCRUM-99) (epic [SCRUM-74](https://rozvaa.atlassian.net/browse/SCRUM-74)) | C1, C3, C15 |

### MVP1 — API contracts (DRAFT)

| Row ID | Item | Detail | Role / permission | Idempotency-Key | Emits / consumes | Epic ID | Story ID | Jira key | Constraints | Mismatch / note | p95 latency target | Payload limit |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| M1-API-05-01 | POST /v1/stock/movements ; GET /v1/stock?warehouse=&item= | Req: {"type":"ISSUE","item_id":"it_1","warehouse":"PUNE-MAIN","qty":50,"ref":{"type":"SO","id":"so_1"}} Resp: {"id":"mv_1","on_hand":150,"available":120} Errors: INSUFFICIENT_STOCK 422, WAREHOUSE_INACTIVE 422 | STORE | Yes (POST) | outbox events per M1-3 | EP-MVP1-05 | ST-MVP1-05-01 | [SCRUM-93](https://rozvaa.atlassian.net/browse/SCRUM-93) (epic [SCRUM-74](https://rozvaa.atlassian.net/browse/SCRUM-74)) | C2, C5, C6, C12 | blocked by [SCRUM-19](https://rozvaa.atlassian.net/browse/SCRUM-19), [SCRUM-20](https://rozvaa.atlassian.net/browse/SCRUM-20) | <500 ms (story ASSUMPTION; MVP0 DoD <300 ms — M1-MM-02) | 1 MB JSON (story) |
| M1-API-05-02 | POST /v1/stock/reservations ; POST /v1/items/{id}/reorder-rule ; GET /v1/batches?expiring_before= | Req: {"order_line_id":"sol_1","item_id":"it_1","qty":100,"batch":"B2610-07"} Resp: {"id":"rs_1","available_after":50} Errors: OVER_RESERVE 422, BATCH_EXPIRED 422 | STORE | Yes (POST) | outbox events per M1-3 | EP-MVP1-05 | ST-MVP1-05-02 | [SCRUM-99](https://rozvaa.atlassian.net/browse/SCRUM-99) (epic [SCRUM-74](https://rozvaa.atlassian.net/browse/SCRUM-74)) | C2, C5, C6, C12 | blocked by [SCRUM-93](https://rozvaa.atlassian.net/browse/SCRUM-93) | <500 ms (story ASSUMPTION; MVP0 DoD <300 ms — M1-MM-02) | 1 MB JSON (story) |

### MVP1 — Events (DRAFT)

| Row ID | Item | Detail | Emits / consumes | Epic ID | Story ID | Jira key | Constraints |
|---|---|---|---|---|---|---|---|
| M1-EVT-04 | stock.moved.v1 | story StockMoved; consumers: sales-crm (MRP), collections (inventory value) | producer: inventory | EP-MVP1-02 | ST-MVP1-02-02 | [SCRUM-107](https://rozvaa.atlassian.net/browse/SCRUM-107) (epic [SCRUM-71](https://rozvaa.atlassian.net/browse/SCRUM-71)) | C4, C6, C12 |

### MVP1 — Feature flags (with OFF behaviour) (DRAFT)

| Row ID | Item | Detail | Epic ID | Story ID | Jira key | Constraints | Mismatch / note | Flag OFF behaviour |
|---|---|---|---|---|---|---|---|---|
| M1-FLG-05-01 | buy.inventory.05_01 | Stock ledger, movements & overview API | EP-MVP1-05 | ST-MVP1-05-01 | [SCRUM-93](https://rozvaa.atlassian.net/browse/SCRUM-93) (epic [SCRUM-74](https://rozvaa.atlassian.net/browse/SCRUM-74)) | C9 | key format <module>.<area>.<NN_NN> differs from MVP0 proposed keys — M1-MM-09 | OFF: hidden / 404 FEATURE_DISABLED; data readable, no writes lost (story) |
| M1-FLG-05-02 | buy.inventory.05_02 | Batches, reservations & reorder-to-PR API | EP-MVP1-05 | ST-MVP1-05-02 | [SCRUM-99](https://rozvaa.atlassian.net/browse/SCRUM-99) (epic [SCRUM-74](https://rozvaa.atlassian.net/browse/SCRUM-74)) | C9 | key format <module>.<area>.<NN_NN> differs from MVP0 proposed keys — M1-MM-09 | OFF: hidden / 404 FEATURE_DISABLED; data readable, no writes lost (story) |
| M1-FLG-05-03 | buy.inventory.05_03 | Stock overview, batches, movements and reorder screens | EP-MVP1-05 | ST-MVP1-05-03 | [SCRUM-115](https://rozvaa.atlassian.net/browse/SCRUM-115) (epic [SCRUM-74](https://rozvaa.atlassian.net/browse/SCRUM-74)) | C9 | key format <module>.<area>.<NN_NN> differs from MVP0 proposed keys — M1-MM-09 | OFF: hidden / 404 FEATURE_DISABLED; data readable, no writes lost (story) |

### MVP1 — Observability (DRAFT)

| Row ID | Item | Detail | Epic ID | Story ID | Jira key | Constraints | Mismatch / note |
|---|---|---|---|---|---|---|---|
| M1-OBS-INVENT | business metrics: bos_stock_movements_total{type} | RED + logs + spans + SLO + alerts + Grafana panel per story (values above) | EP-MVP1-05 | ST-MVP1-05-01, ST-MVP1-05-02, ST-MVP1-05-03 | [SCRUM-93](https://rozvaa.atlassian.net/browse/SCRUM-93), [SCRUM-99](https://rozvaa.atlassian.net/browse/SCRUM-99), [SCRUM-115](https://rozvaa.atlassian.net/browse/SCRUM-115) | C12 | story |

### MVP1 — UI → API (DRAFT)

| Row ID | Item | Detail | Epic ID | Story ID | Jira key | Constraints | Mismatch / note |
|---|---|---|---|---|---|---|---|
| M1-UI-05-03 | Stock overview, batches, movements and reorder screens | Buy MFE. | EP-MVP1-05 | ST-MVP1-05-03 | [SCRUM-115](https://rozvaa.atlassian.net/browse/SCRUM-115) (epic [SCRUM-74](https://rozvaa.atlassian.net/browse/SCRUM-74)) | C9 | blocked by [SCRUM-93](https://rozvaa.atlassian.net/browse/SCRUM-93), [SCRUM-99](https://rozvaa.atlassian.net/browse/SCRUM-99) |
