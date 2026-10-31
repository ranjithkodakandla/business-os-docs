# documents — MVP0/MVP1 design (DRAFT)

> **DRAFT — not frozen.** Design only: no OpenAPI, nothing provisioned. Open decisions OD1–OD9 are **UNCONFIRMED**. Mismatches with the Architect Doc are flagged, not resolved.

Sources: [Sheet tab 'MVP0-1 Design'](https://docs.google.com/spreadsheets/d/18p-aJ0QeW6L3SJ-CNSrNCjwKWBpVdkQAfkzeKOg40Oc/edit#gid=119904232) · [BOS Architect's Doc](https://docs.google.com/document/d/1sKoivvz3JLkKPI1Jta0i6XWnWuW57-rSwWICmm1w9VM/edit) · Jira SCRUM (MVP0 + MVP1 epics SCRUM-70..89, stories SCRUM-90..143). Generated 31 Oct 2026 IST.

Conventions, error/idempotency rules, quality gates, open decisions, mismatches and gaps: see [README](README.md). Rows: MVP0 3, MVP1 9.

## MVP0

### MVP0 — Data model (DRAFT)

| Row ID | Item | Detail | Epic ID | Story ID | Jira key | Constraints |
|---|---|---|---|---|---|---|
| TBL-DC-01 | documents, document_versions | tenant_id; GCS path prefix t/{tenant_id}/; used for import files | EP-MVP0-01 | ST-MVP0-01-07 | [SCRUM-30](https://rozvaa.atlassian.net/browse/SCRUM-30) | C1, C15 |

### MVP0 — Resilience (DRAFT)

| Row ID | Item | Detail | Epic ID | Story ID | Jira key | Constraints | Mismatch / note |
|---|---|---|---|---|---|---|---|
| RES-DC-01 | → GCS | timeout 10 s; 3 retries; breaker; fallback: upload 503, retryable | NO STORY | NO STORY | — | C6, C12 | proposed |

### MVP0 — Observability (DRAFT)

| Row ID | Item | Detail | Epic ID | Story ID | Jira key | Constraints | Mismatch / note |
|---|---|---|---|---|---|---|---|
| OBS-DOCUME | RED + business metrics | RED per endpoint; business: uploads, bytes_stored; JSON logs PII-masked (TST-06) with trace_id, hashed tenant_id; OTel spans incl. DB/Kafka/outbound; SLO: availability 99.9% (proposed unless story); alerts: burn-rate 2%/1h & 5%/6h, DLQ>0, breaker open; Grafana panel "documents" | EP-MVP0-06 | ST-MVP0-06-01, ST-MVP0-06-02 | [SCRUM-22](https://rozvaa.atlassian.net/browse/SCRUM-22), [SCRUM-28](https://rozvaa.atlassian.net/browse/SCRUM-28) | C12 | proposed |

## MVP1

### MVP1 — Data model (DRAFT)

| Row ID | Item | Detail | Epic ID | Story ID | Jira key | Constraints |
|---|---|---|---|---|---|---|
| M1-TBL-DOCU-01 | documents (category, expiry), per-tenant object prefix | tenant_id + RLS policy; story; 5-min signed URLs, virus scan | EP-MVP1-10 | ST-MVP1-10-01 | [SCRUM-120](https://rozvaa.atlassian.net/browse/SCRUM-120) (epic [SCRUM-79](https://rozvaa.atlassian.net/browse/SCRUM-79)) | C1, C3, C15 |
| M1-TBL-DOCU-02 | print_templates, render_jobs | tenant_id + RLS policy; proposed | EP-MVP1-10 | ST-MVP1-10-02 | [SCRUM-121](https://rozvaa.atlassian.net/browse/SCRUM-121) (epic [SCRUM-79](https://rozvaa.atlassian.net/browse/SCRUM-79)) | C1, C3, C15 |

### MVP1 — API contracts (DRAFT)

| Row ID | Item | Detail | Role / permission | Idempotency-Key | Emits / consumes | Epic ID | Story ID | Jira key | Constraints | Mismatch / note | p95 latency target | Payload limit |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| M1-API-10-01 | POST /v1/documents (multipart) ; GET /v1/documents?category=&expiring_before= | Req: category=LICENSE&valid_to=2027-03-31&file=@factory_licence.pdf Resp: {"id":"doc_1","url":"signed...","valid_to":"2027-03-31"} Errors: FILE_TOO_LARGE 413, VIRUS_DETECTED 422 | ADMIN | Yes (POST) | outbox events per M1-3 | EP-MVP1-10 | ST-MVP1-10-01 | [SCRUM-120](https://rozvaa.atlassian.net/browse/SCRUM-120) (epic [SCRUM-79](https://rozvaa.atlassian.net/browse/SCRUM-79)) | C2, C5, C6, C12 | blocked by [SCRUM-24](https://rozvaa.atlassian.net/browse/SCRUM-24), [SCRUM-39](https://rozvaa.atlassian.net/browse/SCRUM-39) | <500 ms (story ASSUMPTION; MVP0 DoD <300 ms — M1-MM-02) | file limit per story (FILE_TOO_LARGE 413) — size not stated |
| M1-API-10-02 | POST /v1/templates ; POST /v1/render | Req: {"doc_type":"TAX_INVOICE","paper":"A4","blocks":[{"type":"header","logo":true},{"type":"lines","columns":["desc","hsn","qty","uom","rate","taxable","gst"]}] Resp: {"id":"tpl_1","valid":true} Errors: TEMPLATE_MISSING_MANDATORY 422 | ADMIN | Yes (POST) | outbox events per M1-3 | EP-MVP1-10 | ST-MVP1-10-02 | [SCRUM-121](https://rozvaa.atlassian.net/browse/SCRUM-121) (epic [SCRUM-79](https://rozvaa.atlassian.net/browse/SCRUM-79)) | C2, C5, C6, C12 |  | <500 ms (story ASSUMPTION; MVP0 DoD <300 ms — M1-MM-02) | 1 MB JSON (story) |

### MVP1 — Feature flags (with OFF behaviour) (DRAFT)

| Row ID | Item | Detail | Epic ID | Story ID | Jira key | Constraints | Mismatch / note | Flag OFF behaviour |
|---|---|---|---|---|---|---|---|---|
| M1-FLG-10-01 | admin.docs.10_01 | Document vault with categories and expiry tracking | EP-MVP1-10 | ST-MVP1-10-01 | [SCRUM-120](https://rozvaa.atlassian.net/browse/SCRUM-120) (epic [SCRUM-79](https://rozvaa.atlassian.net/browse/SCRUM-79)) | C9 | key format <module>.<area>.<NN_NN> differs from MVP0 proposed keys — M1-MM-09 | OFF: hidden / 404 FEATURE_DISABLED; data readable, no writes lost (story) |
| M1-FLG-10-02 | admin.docs.10_02 | Print template designer & renderer | EP-MVP1-10 | ST-MVP1-10-02 | [SCRUM-121](https://rozvaa.atlassian.net/browse/SCRUM-121) (epic [SCRUM-79](https://rozvaa.atlassian.net/browse/SCRUM-79)) | C9 | key format <module>.<area>.<NN_NN> differs from MVP0 proposed keys — M1-MM-09 | OFF: hidden / 404 FEATURE_DISABLED; data readable, no writes lost (story) |
| M1-FLG-10-04 | admin.docs.10_04 | Documents vault, template designer and share/print UI | EP-MVP1-10 | ST-MVP1-10-04 | [SCRUM-140](https://rozvaa.atlassian.net/browse/SCRUM-140) (epic [SCRUM-79](https://rozvaa.atlassian.net/browse/SCRUM-79)) | C9 | key format <module>.<area>.<NN_NN> differs from MVP0 proposed keys — M1-MM-09 | OFF: hidden / 404 FEATURE_DISABLED; data readable, no writes lost (story) |

### MVP1 — Observability (DRAFT)

| Row ID | Item | Detail | Epic ID | Story ID | Jira key | Constraints | Mismatch / note |
|---|---|---|---|---|---|---|---|
| M1-OBS-DOCUME | business metrics: none stated | RED + logs + spans + SLO + alerts + Grafana panel per story (values above) | EP-MVP1-10 | ST-MVP1-10-01, ST-MVP1-10-02, ST-MVP1-10-04 | [SCRUM-120](https://rozvaa.atlassian.net/browse/SCRUM-120), [SCRUM-121](https://rozvaa.atlassian.net/browse/SCRUM-121), [SCRUM-140](https://rozvaa.atlassian.net/browse/SCRUM-140) | C12 | story |

### MVP1 — UI → API (DRAFT)

| Row ID | Item | Detail | Epic ID | Story ID | Jira key | Constraints | Mismatch / note |
|---|---|---|---|---|---|---|---|
| M1-UI-10-04 | Documents vault, template designer and share/print UI | Shell + Money MFE. | EP-MVP1-10 | ST-MVP1-10-04 | [SCRUM-140](https://rozvaa.atlassian.net/browse/SCRUM-140) (epic [SCRUM-79](https://rozvaa.atlassian.net/browse/SCRUM-79)) | C9 | blocked by [SCRUM-120](https://rozvaa.atlassian.net/browse/SCRUM-120), [SCRUM-121](https://rozvaa.atlassian.net/browse/SCRUM-121), [SCRUM-128](https://rozvaa.atlassian.net/browse/SCRUM-128) |
