# notifications — MVP0/MVP1 design (DRAFT)

> **DRAFT — not frozen.** Design only: no OpenAPI, nothing provisioned. Open decisions OD1–OD9 are **UNCONFIRMED**. Mismatches with the Architect Doc are flagged, not resolved.

Sources: [Sheet tab 'MVP0-1 Design'](https://docs.google.com/spreadsheets/d/18p-aJ0QeW6L3SJ-CNSrNCjwKWBpVdkQAfkzeKOg40Oc/edit#gid=119904232) · [BOS Architect's Doc](https://docs.google.com/document/d/1sKoivvz3JLkKPI1Jta0i6XWnWuW57-rSwWICmm1w9VM/edit) · Jira SCRUM (MVP0 + MVP1 epics SCRUM-70..89, stories SCRUM-90..143). Generated 31 Oct 2026 IST.

Conventions, error/idempotency rules, quality gates, open decisions, mismatches and gaps: see [README](README.md). Rows: MVP0 16, MVP1 11.

## MVP0

### MVP0 — Data model (DRAFT)

| Row ID | Item | Detail | Epic ID | Story ID | Jira key | Constraints | Mismatch / note |
|---|---|---|---|---|---|---|---|
| TBL-NT-01 | notifications, notification_preferences | tenant_id, user_id, type, body, read_at; prefs: channels jsonb | EP-MVP0-10 | ST-MVP0-10-01 | [SCRUM-39](https://rozvaa.atlassian.net/browse/SCRUM-39) | C1 |  |
| TBL-NT-02 | templates, messages, deliveries | tenant_id; template: name, channel, status(APPROVED...), DLT id; message: to, template, status | EP-MVP0-14, EP-MVP0-13 | ST-MVP0-14-02, ST-MVP0-13-02 | [SCRUM-69](https://rozvaa.atlassian.net/browse/SCRUM-69), [SCRUM-52](https://rozvaa.atlassian.net/browse/SCRUM-52) | C1, C7 |  |
| TBL-NT-03 | threads (tickets), inbound_messages (webhook inbox) | tenant_id; thread: channel, from, party_id, status, last_inbound_at (24h window) | EP-MVP0-11 | ST-MVP0-11-01 | [SCRUM-40](https://rozvaa.atlassian.net/browse/SCRUM-40) | C1, C7 | Story calls them tickets; Doc calls them threads (MM-11). |
| TBL-NT-04 | consents | tenant_id, id, principal, purpose, notice_version, granted, granted_at — append-only | EP-MVP0-05 | ST-MVP0-05-05 | [SCRUM-36](https://rozvaa.atlassian.net/browse/SCRUM-36) | C1 |  |

### MVP0 — API contracts (DRAFT)

| Row ID | Item | Detail | Role / permission | Idempotency-Key | Emits / consumes | Epic ID | Story ID | Jira key | Constraints | Mismatch / note | p95 latency target | Payload limit |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| API-NT-01 | GET /v1/notifications?cursor=; PUT /v1/notification-preferences | prefs {channels{...}} → {saved} | self | PUT Yes | — | EP-MVP0-10 | ST-MVP0-10-01 | [SCRUM-39](https://rozvaa.atlassian.net/browse/SCRUM-39) | C5 |  | <300 ms @50 rps (story DoD) | 1 MB (proposed) |
| API-NT-02 | POST /webhooks/whatsapp | Meta payload, X-Hub-Signature-256; SIGNATURE_INVALID 401; tenant resolved from WABA phone-number id → catalog, never from payload field | Meta signature | dedupe on Meta message id | inbound.received.v1 | EP-MVP0-11 | ST-MVP0-11-01 | [SCRUM-40](https://rozvaa.atlassian.net/browse/SCRUM-40) | C2, C5, C7 | Story path is unversioned; Doc says /v1/hooks/{provider} (MM-04). | <300 ms @50 rps (story DoD); ack <1 s to Meta (proposed) | 256 KB (proposed) |
| API-NT-03 | GET /v1/tickets?status=&cursor= | → {items[{id, channel, from, party_id, status}]} | SALES/ADMIN | n/a | — | EP-MVP0-11 | ST-MVP0-11-01 | [SCRUM-40](https://rozvaa.atlassian.net/browse/SCRUM-40) | C5 | Doc calls these /v1/threads (MM-11). | <300 ms @50 rps (story DoD) | 1 MB (proposed) |
| API-NT-04 | POST /v1/tickets/{id}/draft | {tone, lang} → {draft, confidence} | SALES | Yes | — | EP-MVP0-11 | ST-MVP0-11-02 | [SCRUM-50](https://rozvaa.atlassian.net/browse/SCRUM-50) | C13 | Drafting is AI → ai-agents should own logic; path sits in notifications (MM-11). | <300 ms @50 rps (story DoD) | 1 MB (proposed) |
| API-NT-05 | POST /v1/consents | {principal, purpose, notice_version, granted} → {id, granted_at} | system (on signup) / data principal | Yes | consent.changed.v1 | EP-MVP0-05 | ST-MVP0-05-05 | [SCRUM-36](https://rozvaa.atlassian.net/browse/SCRUM-36) | C2 | Consent of users vs customers in one table, both in notifications — confirm. | <300 ms @50 rps (story DoD) | 1 MB (proposed) |

### MVP0 — Events (DRAFT)

| Row ID | Item | Detail | Emits / consumes | Epic ID | Story ID | Jira key | Constraints |
|---|---|---|---|---|---|---|---|
| EVT-11 | inbound.received.v1 | consumers: ai-agents (triage), audit; data: thread_id, channel, party_id | producer: notifications | EP-MVP0-11 | ST-MVP0-11-01 | [SCRUM-40](https://rozvaa.atlassian.net/browse/SCRUM-40) | C4, C6, C12 |
| EVT-12 | message.delivered.v1 / message.failed.v1 | consumers: collections-payments, purchase, audit; data: message_id, status, correlation_id | producer: notifications | EP-MVP0-13 | ST-MVP0-13-02 | [SCRUM-52](https://rozvaa.atlassian.net/browse/SCRUM-52) | C4, C6, C12 |
| EVT-13 | consent.changed.v1 | consumers: audit; data: consent_id, purpose, granted | producer: notifications | EP-MVP0-05 | ST-MVP0-05-05 | [SCRUM-36](https://rozvaa.atlassian.net/browse/SCRUM-36) | C4, C6, C12 |

### MVP0 — Resilience (DRAFT)

| Row ID | Item | Detail | Epic ID | Story ID | Jira key | Constraints | Mismatch / note |
|---|---|---|---|---|---|---|---|
| RES-NT-01 | → WhatsApp Cloud API | timeout 5 s; 3 retries (idempotent by message_id); breaker 60 s; bulkhead per tenant; fallback: SMS/email fallback per preference; message.failed.v1 | EP-MVP0-10, EP-MVP0-13 | ST-MVP0-10-01, ST-MVP0-13-02 | [SCRUM-39](https://rozvaa.atlassian.net/browse/SCRUM-39), [SCRUM-52](https://rozvaa.atlassian.net/browse/SCRUM-52) | C6, C12 | proposed |
| RES-NT-02 | → SMS (DLT) / email provider | timeout 3 s; 3 retries; breaker; fallback: queue for retry; in-app feed always written | EP-MVP0-10 | ST-MVP0-10-01 | [SCRUM-39](https://rozvaa.atlassian.net/browse/SCRUM-39) | C6, C12 | proposed |

### MVP0 — Observability (DRAFT)

| Row ID | Item | Detail | Epic ID | Story ID | Jira key | Constraints | Mismatch / note |
|---|---|---|---|---|---|---|---|
| OBS-NOTIFI | RED + business metrics | RED per endpoint; business: delivered_rate per channel, webhook_lag_s, template_rejections; JSON logs PII-masked (TST-06) with trace_id, hashed tenant_id; OTel spans incl. DB/Kafka/outbound; SLO: delivery ≥98%; webhook ack <1 s (proposed unless story); alerts: burn-rate 2%/1h & 5%/6h, DLQ>0, breaker open; Grafana panel "notifications" | EP-MVP0-06 | ST-MVP0-06-01, ST-MVP0-06-02 | [SCRUM-22](https://rozvaa.atlassian.net/browse/SCRUM-22), [SCRUM-28](https://rozvaa.atlassian.net/browse/SCRUM-28) | C12 | proposed |

### MVP0 — Infra (DRAFT)

| Row ID | Item | Detail | Epic ID | Story ID | Jira key | Constraints | Mismatch / note |
|---|---|---|---|---|---|---|---|
| INF-05 | WABA verification, templates, DLT | Templates table + approval status only; DLT/WABA registration is ops, not designed | EP-MVP0-14 | ST-MVP0-14-02 | [SCRUM-69](https://rozvaa.atlassian.net/browse/SCRUM-69) | C7 | Partial coverage (GAP-S5) |

## MVP1

### MVP1 — Data model (DRAFT)

| Row ID | Item | Detail | Epic ID | Story ID | Jira key | Constraints |
|---|---|---|---|---|---|---|
| M1-TBL-NOTI-01 | alert_catalogue, alert_preferences | tenant_id + RLS policy; proposed | EP-MVP1-17 | ST-MVP1-17-01 | [SCRUM-106](https://rozvaa.atlassian.net/browse/SCRUM-106) (epic [SCRUM-86](https://rozvaa.atlassian.net/browse/SCRUM-86)) | C1, C3, C15 |
| M1-TBL-NOTI-02 | invoice_shares | tenant_id + RLS policy; proposed; needs wa_optin | EP-MVP1-10 | ST-MVP1-10-03 | [SCRUM-128](https://rozvaa.atlassian.net/browse/SCRUM-128) (epic [SCRUM-79](https://rozvaa.atlassian.net/browse/SCRUM-79)) | C1, C3, C15 |

### MVP1 — API contracts (DRAFT)

| Row ID | Item | Detail | Role / permission | Idempotency-Key | Emits / consumes | Epic ID | Story ID | Jira key | Constraints | Mismatch / note | p95 latency target | Payload limit |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| M1-API-17-01 | GET /v1/alerts/catalogue ; PUT /v1/me/alert-preferences | Req: {"alert":"INVOICE_OVERDUE","channels":["IN_APP","WHATSAPP"],"frequency":"DIGEST_DAILY","scope":"OWN"} Resp: {"saved":true} Errors: SCOPE_EXCEEDS_ROLE 422, CHANNEL_NOT_ALLOWED 422 | per story | n/a | outbox events per M1-3 | EP-MVP1-17 | ST-MVP1-17-01 | [SCRUM-106](https://rozvaa.atlassian.net/browse/SCRUM-106) (epic [SCRUM-86](https://rozvaa.atlassian.net/browse/SCRUM-86)) | C2, C5, C6, C12 | blocked by [SCRUM-39](https://rozvaa.atlassian.net/browse/SCRUM-39), [SCRUM-49](https://rozvaa.atlassian.net/browse/SCRUM-49) | <500 ms (story ASSUMPTION; MVP0 DoD <300 ms — M1-MM-02) | 1 MB JSON (story) |
| M1-API-10-03 | POST /v1/invoices/{id}/share | Req: {"channel":"WHATSAPP","include_upi":true} Resp: {"message_id":"wamid...","status":"SENT"} Errors: NO_OPTIN 422, TEMPLATE_NOT_APPROVED 422 | SALES | Yes (POST) | outbox events per M1-3 | EP-MVP1-10 | ST-MVP1-10-03 | [SCRUM-128](https://rozvaa.atlassian.net/browse/SCRUM-128) (epic [SCRUM-79](https://rozvaa.atlassian.net/browse/SCRUM-79)) | C2, C5, C6, C12 |  | <500 ms (story ASSUMPTION; MVP0 DoD <300 ms — M1-MM-02) | 1 MB JSON (story) |

### MVP1 — Feature flags (with OFF behaviour) (DRAFT)

| Row ID | Item | Detail | Epic ID | Story ID | Jira key | Constraints | Mismatch / note | Flag OFF behaviour |
|---|---|---|---|---|---|---|---|---|
| M1-FLG-17-01 | home.alerts.17_01 | Alert catalogue & preference engine (channel, frequency, scope per RBAC) | EP-MVP1-17 | ST-MVP1-17-01 | [SCRUM-106](https://rozvaa.atlassian.net/browse/SCRUM-106) (epic [SCRUM-86](https://rozvaa.atlassian.net/browse/SCRUM-86)) | C9 | key format <module>.<area>.<NN_NN> differs from MVP0 proposed keys — M1-MM-09 | OFF: hidden / 404 FEATURE_DISABLED; data readable, no writes lost (story) |
| M1-FLG-17-02 | home.alerts.17_02 | Alert preferences screen | EP-MVP1-17 | ST-MVP1-17-02 | [SCRUM-117](https://rozvaa.atlassian.net/browse/SCRUM-117) (epic [SCRUM-86](https://rozvaa.atlassian.net/browse/SCRUM-86)) | C9 | key format <module>.<area>.<NN_NN> differs from MVP0 proposed keys — M1-MM-09 | OFF: hidden / 404 FEATURE_DISABLED; data readable, no writes lost (story) |
| M1-FLG-10-03 | admin.docs.10_03 | One-tap invoice share on WhatsApp + UPI link | EP-MVP1-10 | ST-MVP1-10-03 | [SCRUM-128](https://rozvaa.atlassian.net/browse/SCRUM-128) (epic [SCRUM-79](https://rozvaa.atlassian.net/browse/SCRUM-79)) | C9 | key format <module>.<area>.<NN_NN> differs from MVP0 proposed keys — M1-MM-09 | OFF: hidden / 404 FEATURE_DISABLED; data readable, no writes lost (story) |

### MVP1 — Resilience (DRAFT)

| Row ID | Item | Detail | Epic ID | Story ID | Jira key | Constraints | Mismatch / note |
|---|---|---|---|---|---|---|---|
| M1-RES-17-01 | cb.messaging_gateway | timeout 10 s external; breaker 50%/20 calls, half-open 30 s; bulkhead 10/tenant; fallback: degrade to manual / template text, never block the core flow | EP-MVP1-17 | ST-MVP1-17-01 | [SCRUM-106](https://rozvaa.atlassian.net/browse/SCRUM-106) (epic [SCRUM-86](https://rozvaa.atlassian.net/browse/SCRUM-86)) | C6, C12 | story |
| M1-RES-10-03 | cb.whatsapp_cloud_api | timeout 10 s external; breaker 50%/20 calls, half-open 30 s; bulkhead 10/tenant; fallback: degrade to manual / template text, never block the core flow | EP-MVP1-10 | ST-MVP1-10-03 | [SCRUM-128](https://rozvaa.atlassian.net/browse/SCRUM-128) (epic [SCRUM-79](https://rozvaa.atlassian.net/browse/SCRUM-79)) | C6, C12 | story |

### MVP1 — Observability (DRAFT)

| Row ID | Item | Detail | Epic ID | Story ID | Jira key | Constraints | Mismatch / note |
|---|---|---|---|---|---|---|---|
| M1-OBS-NOTIFI | business metrics: bos_invoice_shares_total{channel,status} | RED + logs + spans + SLO + alerts + Grafana panel per story (values above) | EP-MVP1-10, EP-MVP1-17 | ST-MVP1-17-01, ST-MVP1-17-02, ST-MVP1-10-03 | [SCRUM-106](https://rozvaa.atlassian.net/browse/SCRUM-106), [SCRUM-117](https://rozvaa.atlassian.net/browse/SCRUM-117), [SCRUM-128](https://rozvaa.atlassian.net/browse/SCRUM-128) | C12 | story |

### MVP1 — UI → API (DRAFT)

| Row ID | Item | Detail | Epic ID | Story ID | Jira key | Constraints |
|---|---|---|---|---|---|---|
| M1-UI-17-02 | Alert preferences screen | Shell settings. | EP-MVP1-17 | ST-MVP1-17-02 | [SCRUM-117](https://rozvaa.atlassian.net/browse/SCRUM-117) (epic [SCRUM-86](https://rozvaa.atlassian.net/browse/SCRUM-86)) | C9 |
