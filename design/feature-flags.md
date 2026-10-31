# feature-flags — MVP0/MVP1 design (DRAFT)

> **DRAFT — not frozen.** Design only: no OpenAPI, nothing provisioned. Open decisions OD1–OD9 are **UNCONFIRMED**. Mismatches with the Architect Doc are flagged, not resolved.

Sources: [Sheet tab 'MVP0-1 Design'](https://docs.google.com/spreadsheets/d/18p-aJ0QeW6L3SJ-CNSrNCjwKWBpVdkQAfkzeKOg40Oc/edit#gid=119904232) · [BOS Architect's Doc](https://docs.google.com/document/d/1sKoivvz3JLkKPI1Jta0i6XWnWuW57-rSwWICmm1w9VM/edit) · Jira SCRUM (MVP0 + MVP1 epics SCRUM-70..89, stories SCRUM-90..143). Generated 31 Oct 2026 IST.

Conventions, error/idempotency rules, quality gates, open decisions, mismatches and gaps: see [README](README.md). Rows: MVP0 20, MVP1 0.

## MVP0

### MVP0 — Data model (DRAFT)

| Row ID | Item | Detail | Epic ID | Story ID | Jira key | Constraints |
|---|---|---|---|---|---|---|
| TBL-FF-01 | flags, flag_stages, segments (GLOBAL) | key, stage(INTERNAL\|ALPHA\|BETA\|PERCENT\|GA), killed, kill_reason — platform-owned, allowlisted | EP-MVP0-03 | ST-MVP0-03-01, ST-MVP0-03-02 | [SCRUM-25](https://rozvaa.atlassian.net/browse/SCRUM-25), [SCRUM-34](https://rozvaa.atlassian.net/browse/SCRUM-34) | C1, C9 |
| TBL-FF-02 | overrides, flag_audit | overrides: tenant_id, key, on; flag_audit append-only | EP-MVP0-03 | ST-MVP0-03-01, ST-MVP0-03-03 | [SCRUM-25](https://rozvaa.atlassian.net/browse/SCRUM-25), [SCRUM-45](https://rozvaa.atlassian.net/browse/SCRUM-45) | C1 |

### MVP0 — API contracts (DRAFT)

| Row ID | Item | Detail | Role / permission | Idempotency-Key | Emits / consumes | Epic ID | Story ID | Jira key | Constraints | Mismatch / note | p95 latency target | Payload limit |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| API-FF-01 | POST /v1/flags/evaluate | {keys[], context{unit_id}} → {flags{key:{on, variant\|reason}}, ttl_s 30}. FLAG_UNKNOWN → on:false; RATE_LIMITED 429 | any authenticated | n/a (read) | — | EP-MVP0-03 | ST-MVP0-03-01 | [SCRUM-25](https://rozvaa.atlassian.net/browse/SCRUM-25) | C9 | Doc: GET /v1/flags/evaluate; Arch tab: /v1/evaluate (MM-03). | <5 ms from SDK cache (story 03-01); API <300 ms @50 rps (story DoD) | 64 KB (proposed) |
| API-FF-02 | POST /v1/flags/{key}/kill | {reason, scope ALL\|tenant} → {key, killed, effective_within_s 30}. NEEDS_SECOND_APPROVER 409 | PLATFORM_ADMIN + 2nd approver | Yes | flags.flag.killed.v1 | EP-MVP0-03 | ST-MVP0-03-02 | [SCRUM-34](https://rozvaa.atlassian.net/browse/SCRUM-34) | C1, C9 | Kill SLA conflict: story 30 s, epic <60 s, earlier sheet <10 s (MM-12). Scope 'tenant' would need a target tenant in the body → conflicts with C2 for platform ops (MM-12). | <300 ms @50 rps (story DoD) | 1 MB (proposed) |

### MVP0 — Events (DRAFT)

| Row ID | Item | Detail | Emits / consumes | Epic ID | Story ID | Jira key | Constraints |
|---|---|---|---|---|---|---|---|
| EVT-16 | flags.flag.changed.v1 / flags.flag.killed.v1 | consumers: all services (Valkey pub/sub fan-out), audit; data: key, scope, reason | producer: feature-flags | EP-MVP0-03 | ST-MVP0-03-02 | [SCRUM-34](https://rozvaa.atlassian.net/browse/SCRUM-34) | C4, C6, C12 |

### MVP0 — Feature flags (with OFF behaviour) (DRAFT)

| Row ID | Item | Detail | Epic ID | Story ID | Jira key | Constraints | Mismatch / note | p95 latency target |
|---|---|---|---|---|---|---|---|---|
| FLG-01 | home.chat | rollout flag | EP-MVP0-09 | ST-MVP0-09-01, ST-MVP0-09-04 | [SCRUM-38](https://rozvaa.atlassian.net/browse/SCRUM-38), [SCRUM-62](https://rozvaa.atlassian.net/browse/SCRUM-62) | C9 | from story | Home shows Needs-you-today cards without chat; chat history kept read-only |
| FLG-02 | collections.whatsapp | rollout flag | EP-MVP0-13 | ST-MVP0-13-02 | [SCRUM-52](https://rozvaa.atlassian.net/browse/SCRUM-52) | C9 | from story | Follow-ups fall back to manual/email; scheduled WhatsApp follow-ups paused, not deleted |
| FLG-03 | collections.sprint | rollout flag | EP-MVP0-13 | ST-MVP0-13-01, ST-MVP0-13-04 | [SCRUM-42](https://rozvaa.atlassian.net/browse/SCRUM-42), [SCRUM-67](https://rozvaa.atlassian.net/browse/SCRUM-67) | C9 | proposed | Sprint board hidden; receivables list still served; sprint rows retained |
| FLG-04 | inbox.tickets | rollout flag | EP-MVP0-11 | ST-MVP0-11-01, ST-MVP0-11-03 | [SCRUM-40](https://rozvaa.atlassian.net/browse/SCRUM-40), [SCRUM-64](https://rozvaa.atlassian.net/browse/SCRUM-64) | C9 | proposed | Inbox list hidden; inbound messages still stored by webhook |
| FLG-05 | inbox.ai_draft | rollout flag | EP-MVP0-11 | ST-MVP0-11-02 | [SCRUM-50](https://rozvaa.atlassian.net/browse/SCRUM-50) | C9 | proposed | Draft panel hidden; agent types reply manually; drafts retained |
| FLG-06 | purchase.sourcing_agent | rollout flag | EP-MVP0-12 | ST-MVP0-12-03 | [SCRUM-65](https://rozvaa.atlassian.net/browse/SCRUM-65) | C9 | proposed | Sourcing agent hidden; manual RFQ still works; RFQs retained |
| FLG-07 | inventory.bom_check | rollout flag | EP-MVP0-12 | ST-MVP0-12-02 | [SCRUM-51](https://rozvaa.atlassian.net/browse/SCRUM-51) | C9 | proposed | BOM check hidden; no auto PO drafts; existing drafts stay editable |
| FLG-08 | approvals.queue | rollout flag | EP-MVP0-08 | ST-MVP0-08-01, ST-MVP0-08-02 | [SCRUM-37](https://rozvaa.atlassian.net/browse/SCRUM-37), [SCRUM-61](https://rozvaa.atlassian.net/browse/SCRUM-61) | C9 | proposed | Approval-required actions stay pending (never auto-executed); queue data kept |
| FLG-09 | setup.quick_wizard | rollout flag | EP-MVP0-01 | ST-MVP0-01-08, ST-MVP0-01-07 | [SCRUM-55](https://rozvaa.atlassian.net/browse/SCRUM-55), [SCRUM-30](https://rozvaa.atlassian.net/browse/SCRUM-30) | C9 | proposed | Full settings forms instead of wizard; saved progress kept |
| FLG-10 | notifications.compliance_calendar | rollout flag | EP-MVP0-10 | ST-MVP0-10-02, ST-MVP0-10-03 | [SCRUM-49](https://rozvaa.atlassian.net/browse/SCRUM-49), [SCRUM-63](https://rozvaa.atlassian.net/browse/SCRUM-63) | C9 | proposed | Calendar hidden; feed + reminders still sent |
| FLG-11 | integrations.hub | rollout flag | EP-MVP0-14 | ST-MVP0-14-01, ST-MVP0-14-03 | [SCRUM-54](https://rozvaa.atlassian.net/browse/SCRUM-54), [SCRUM-68](https://rozvaa.atlassian.net/browse/SCRUM-68) | C9 | proposed | Hub hidden; existing connections keep syncing |
| FLG-12 | mfe.<module> (one per MFE) | rollout flag | EP-MVP0-07 | ST-MVP0-07-01 | [SCRUM-46](https://rozvaa.atlassian.net/browse/SCRUM-46) | C9 | proposed | Shell renders 'module unavailable' tile; route returns 404 page, data untouched |
| FLG-13 | auth.view_as | rollout flag | EP-MVP0-02 | ST-MVP0-02-06 | [SCRUM-58](https://rozvaa.atlassian.net/browse/SCRUM-58) | C9 | proposed | View-as button hidden; active sessions expire normally |

### MVP0 — Observability (DRAFT)

| Row ID | Item | Detail | Epic ID | Story ID | Jira key | Constraints | Mismatch / note |
|---|---|---|---|---|---|---|---|
| OBS-FEATUR | RED + business metrics | RED per endpoint; business: evaluations, cache_hit_rate, kill_propagation_s; JSON logs PII-masked (TST-06) with trace_id, hashed tenant_id; OTel spans incl. DB/Kafka/outbound; SLO: eval p95 <5 ms SDK; kill ≤30 s (story values); alerts: burn-rate 2%/1h & 5%/6h, DLQ>0, breaker open; Grafana panel "feature-flags" | EP-MVP0-06 | ST-MVP0-06-01, ST-MVP0-06-02 | [SCRUM-22](https://rozvaa.atlassian.net/browse/SCRUM-22), [SCRUM-28](https://rozvaa.atlassian.net/browse/SCRUM-28) | C12 | story (03-01, 03-02) |

### MVP0 — Tests (DRAFT)

| Row ID | Item | Detail | Epic ID | Story ID | Jira key | Constraints |
|---|---|---|---|---|---|---|
| TST-08 | Kill switch | flag killed → all instances off within SLA (MM-12) measured end-to-end | EP-MVP0-03 | ST-MVP0-03-02 | [SCRUM-34](https://rozvaa.atlassian.net/browse/SCRUM-34) | C9 |
