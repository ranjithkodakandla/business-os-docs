# ai-agents — MVP0/MVP1 design (DRAFT)

> **DRAFT — not frozen.** Design only: no OpenAPI, nothing provisioned. Open decisions OD1–OD9 are **UNCONFIRMED**. Mismatches with the Architect Doc are flagged, not resolved.

Sources: [Sheet tab 'MVP0-1 Design'](https://docs.google.com/spreadsheets/d/18p-aJ0QeW6L3SJ-CNSrNCjwKWBpVdkQAfkzeKOg40Oc/edit#gid=119904232) · [BOS Architect's Doc](https://docs.google.com/document/d/1sKoivvz3JLkKPI1Jta0i6XWnWuW57-rSwWICmm1w9VM/edit) · Jira SCRUM (MVP0 + MVP1 epics SCRUM-70..89, stories SCRUM-90..143). Generated 31 Oct 2026 IST.

Conventions, error/idempotency rules, quality gates, open decisions, mismatches and gaps: see [README](README.md). Rows: MVP0 14, MVP1 13.

## MVP0

### MVP0 — Data model (DRAFT)

| Row ID | Item | Detail | Epic ID | Story ID | Jira key | Constraints | Mismatch / note |
|---|---|---|---|---|---|---|---|
| TBL-AI-01 | conversations, runs, run_steps, llm_usage, budgets | tenant_id; llm_usage tokens/cost per tenant via LiteLLM | EP-MVP0-09 | ST-MVP0-09-01 | [SCRUM-38](https://rozvaa.atlassian.net/browse/SCRUM-38) | C1, C13, C15 |  |
| TBL-AI-02 | agent_settings (permissions) | tenant_id, agent, allow[], require_approval[] | EP-MVP0-09 | ST-MVP0-09-02 | [SCRUM-47](https://rozvaa.atlassian.net/browse/SCRUM-47) | C1, C13 |  |
| TBL-AI-03 | approval_rules, approvals (proposals) | rules: doc_type, condition jsonb (amount_gt_paise), approver_role, levels; approvals: status, decision, reason, decided_by | EP-MVP0-08 | ST-MVP0-08-01 | [SCRUM-37](https://rozvaa.atlassian.net/browse/SCRUM-37) | C1, C13, C15 | Approval ownership (ai-agents per Arch tab) is an assumption (MM-08). |
| TBL-AI-04 | home_summary (read model) | tenant_id, needs_you jsonb, glance receivable_paise/payable_paise/cash_paise; fed by events | EP-MVP0-09 | ST-MVP0-09-03 | [SCRUM-48](https://rozvaa.atlassian.net/browse/SCRUM-48) | C1, C4, C14 |  |

### MVP0 — API contracts (DRAFT)

| Row ID | Item | Detail | Role / permission | Idempotency-Key | Emits / consumes | Epic ID | Story ID | Jira key | Constraints | Mismatch / note | p95 latency target | Payload limit |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| API-AI-01 | POST /v1/ai/chat (SSE) | {conversation_id, message, lang} → SSE token…done{conversation_id, citations, actions}. AI_QUOTA_EXCEEDED 429, AI_PROVIDER_DOWN 503, GUARDRAIL_BLOCKED 422 | any; retrieval filtered by caller permissions | Yes | agent.run_completed.v1 | EP-MVP0-09 | ST-MVP0-09-01 | [SCRUM-38](https://rozvaa.atlassian.net/browse/SCRUM-38) | C13, C15 | Doc path /v1/chat (MM-07). Retrieval must call service APIs, not read schemas (C4, C13). | first token <2 s, p95 <300 ms non-stream (proposed for SSE) | 32 KB prompt (proposed) |
| API-AI-02 | PUT /v1/agents/permissions | {agent, allow[], require_approval[]} → {saved} | OWNER | Yes | agent.settings.changed.v1 | EP-MVP0-09 | ST-MVP0-09-02 | [SCRUM-47](https://rozvaa.atlassian.net/browse/SCRUM-47) | C13 |  | <300 ms @50 rps (story DoD) | 1 MB (proposed) |
| API-AI-03 | GET /v1/home/summary | → {needs_you[], glance{receivable, payable, cash}} | any (sections filtered) | n/a | — | EP-MVP0-09 | ST-MVP0-09-03 | [SCRUM-48](https://rozvaa.atlassian.net/browse/SCRUM-48) | C4, C14, C15 | glance amounts unit unstated (MM-06); owner service not in Doc. | <300 ms @50 rps (story DoD) | 1 MB (proposed) |
| API-AI-04 | POST /v1/approval-rules; POST /v1/approvals/{id}/decision | rule {doc_type, condition{amount_gt}, approver_role, levels} → {id}; decision {decision, reason} → {id, status}. NOT_APPROVER 403, ALREADY_DECIDED 409, REASON_REQUIRED 422 | per rule | Yes | agent.action_approved.v1 / approval.decided.v1 | EP-MVP0-08 | ST-MVP0-08-01 | [SCRUM-37](https://rozvaa.atlassian.net/browse/SCRUM-37) | C13, C15 | Doc: /v1/proposals/{id}/approve (MM-08). amount_gt unit (MM-06). | <300 ms @50 rps (story DoD) | 1 MB (proposed) |

### MVP0 — Events (DRAFT)

| Row ID | Item | Detail | Emits / consumes | Epic ID | Story ID | Jira key | Constraints |
|---|---|---|---|---|---|---|---|
| EVT-14 | agent.proposal_created.v1 | consumers: notifications (approver), audit; data: approval_id, doc_type, amount_paise | producer: ai-agents | EP-MVP0-08 | ST-MVP0-08-01 | [SCRUM-37](https://rozvaa.atlassian.net/browse/SCRUM-37) | C4, C6, C12 |
| EVT-15 | agent.action_approved.v1 (approval.decided.v1) | consumers: owning service executes, notifications, audit; data: approval_id, decision, reason | producer: ai-agents | EP-MVP0-08, EP-MVP0-09 | ST-MVP0-08-01, ST-MVP0-09-02 | [SCRUM-37](https://rozvaa.atlassian.net/browse/SCRUM-37), [SCRUM-47](https://rozvaa.atlassian.net/browse/SCRUM-47) | C4, C6, C12 |

### MVP0 — Resilience (DRAFT)

| Row ID | Item | Detail | Epic ID | Story ID | Jira key | Constraints | Mismatch / note |
|---|---|---|---|---|---|---|---|
| RES-AI-01 | → LiteLLM / model provider | timeout 30 s stream, 10 s first token; 1 retry; breaker 60 s; bulkhead per tenant; fallback: secondary model; else canned "AI unavailable" + manual path | EP-MVP0-09 | ST-MVP0-09-01 | [SCRUM-38](https://rozvaa.atlassian.net/browse/SCRUM-38) | C6, C12 | proposed |
| RES-AI-02 | → domain services (tool calls with user token) | timeout 2 s; 1 retry GET only; breaker per service; fallback: answer excludes that section and says so | EP-MVP0-09 | ST-MVP0-09-03 | [SCRUM-48](https://rozvaa.atlassian.net/browse/SCRUM-48) | C6, C12 | proposed |

### MVP0 — Observability (DRAFT)

| Row ID | Item | Detail | Epic ID | Story ID | Jira key | Constraints | Mismatch / note |
|---|---|---|---|---|---|---|---|
| OBS-AI | RED + business metrics | RED per endpoint; business: chat_turns, first_token_ms, eval_score, approvals_pending, tokens_cost; JSON logs PII-masked (TST-06) with trace_id, hashed tenant_id; OTel spans incl. DB/Kafka/outbound; SLO: first token <2 s p95 (proposed unless story); alerts: burn-rate 2%/1h & 5%/6h, DLQ>0, breaker open; Grafana panel "ai-agents" | EP-MVP0-06 | ST-MVP0-06-01, ST-MVP0-06-02 | [SCRUM-22](https://rozvaa.atlassian.net/browse/SCRUM-22), [SCRUM-28](https://rozvaa.atlassian.net/browse/SCRUM-28) | C12 | proposed |

### MVP0 — Tests (DRAFT)

| Row ID | Item | Detail | Epic ID | Story ID | Jira key | Constraints |
|---|---|---|---|---|---|---|
| TST-09 | AI eval | 200-prompt eval ≥85%; no answer cites data the user can't see; all AI writes audited + approval above limit | EP-MVP0-09 | ST-MVP0-09-01, ST-MVP0-09-02 | [SCRUM-38](https://rozvaa.atlassian.net/browse/SCRUM-38), [SCRUM-47](https://rozvaa.atlassian.net/browse/SCRUM-47) | C13, C15 |

## MVP1

### MVP1 — Data model (DRAFT)

| Row ID | Item | Detail | Epic ID | Story ID | Jira key | Constraints |
|---|---|---|---|---|---|---|
| M1-TBL-AI-01 | narratives (grounding result) | tenant_id + RLS policy; proposed | EP-MVP1-09 | ST-MVP1-09-02 | [SCRUM-119](https://rozvaa.atlassian.net/browse/SCRUM-119) (epic [SCRUM-78](https://rozvaa.atlassian.net/browse/SCRUM-78)) | C1, C3, C15 |
| M1-TBL-AI-02 | wa_agent_bindings (mobile↔user via OTP), agent_action_log | tenant_id + RLS policy; proposed | EP-MVP1-19 | ST-MVP1-19-01 | [SCRUM-124](https://rozvaa.atlassian.net/browse/SCRUM-124) (epic [SCRUM-88](https://rozvaa.atlassian.net/browse/SCRUM-88)) | C1, C3, C15 |
| M1-TBL-AI-03 | asr_eval_sets, wer_results (encrypted audio, retention) | tenant_id + RLS policy; proposed | EP-MVP1-19 | ST-MVP1-19-02 | [SCRUM-131](https://rozvaa.atlassian.net/browse/SCRUM-131) (epic [SCRUM-88](https://rozvaa.atlassian.net/browse/SCRUM-88)) | C1, C3, C15 |

### MVP1 — API contracts (DRAFT)

| Row ID | Item | Detail | Role / permission | Idempotency-Key | Emits / consumes | Epic ID | Story ID | Jira key | Constraints | Mismatch / note | p95 latency target | Payload limit |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| M1-API-09-02 | POST /v1/reports/narrative | Req: {"reports":["dso","billing_vs_collections"],"lang":"hi"} Resp: {"text":"...","claims":[{"value":37.5,"source":"dso"}]} Errors: NARRATIVE_UNGROUNDED 422 | OWNER | Yes (POST) | outbox events per M1-3 | EP-MVP1-09 | ST-MVP1-09-02 | [SCRUM-119](https://rozvaa.atlassian.net/browse/SCRUM-119) (epic [SCRUM-78](https://rozvaa.atlassian.net/browse/SCRUM-78)) | C2, C5, C6, C12 |  | <500 ms (story ASSUMPTION; MVP0 DoD <300 ms — M1-MM-02) | 1 MB JSON (story) |
| M1-API-19-01 | POST /webhooks/whatsapp -> POST /v1/agent/actions | Req: {"from":"+919876543210","text":"record 60000 from Shree Ganesh by NEFT"} Resp: {"action":"RECORD_PAYMENT","needs_confirm":true,"preview":"Record Rs 60,000 NEFT from Shree Ganesh Traders?"} Errors: UNKNOWN_SENDER 403, OUTSIDE_WINDOW (template) | per story | Yes (POST) | outbox events per M1-3 | EP-MVP1-19 | ST-MVP1-19-01 | [SCRUM-124](https://rozvaa.atlassian.net/browse/SCRUM-124) (epic [SCRUM-88](https://rozvaa.atlassian.net/browse/SCRUM-88)) | C2, C5, C6, C12 | unversioned webhook (C5) — M1-MM-06; blocked by [SCRUM-38](https://rozvaa.atlassian.net/browse/SCRUM-38), [SCRUM-47](https://rozvaa.atlassian.net/browse/SCRUM-47), [SCRUM-40](https://rozvaa.atlassian.net/browse/SCRUM-40), [SCRUM-106](https://rozvaa.atlassian.net/browse/SCRUM-106) | <500 ms (story ASSUMPTION; MVP0 DoD <300 ms — M1-MM-02) | 1 MB JSON (story) |

### MVP1 — Feature flags (with OFF behaviour) (DRAFT)

| Row ID | Item | Detail | Epic ID | Story ID | Jira key | Constraints | Mismatch / note | Flag OFF behaviour |
|---|---|---|---|---|---|---|---|---|
| M1-FLG-09-02 | money.reports.09_02 | "What the numbers say" AI narrative | EP-MVP1-09 | ST-MVP1-09-02 | [SCRUM-119](https://rozvaa.atlassian.net/browse/SCRUM-119) (epic [SCRUM-78](https://rozvaa.atlassian.net/browse/SCRUM-78)) | C9 | key format <module>.<area>.<NN_NN> differs from MVP0 proposed keys — M1-MM-09 | OFF: hidden / 404 FEATURE_DISABLED; data readable, no writes lost (story) |
| M1-FLG-19-01 | home.wa_agent.19_01 | WhatsApp-native agent actions | EP-MVP1-19 | ST-MVP1-19-01 | [SCRUM-124](https://rozvaa.atlassian.net/browse/SCRUM-124) (epic [SCRUM-88](https://rozvaa.atlassian.net/browse/SCRUM-88)) | C9 | key format <module>.<area>.<NN_NN> differs from MVP0 proposed keys — M1-MM-09 | OFF: hidden / 404 FEATURE_DISABLED; data readable, no writes lost (story) |
| M1-FLG-19-02 | home.wa_agent.19_02 | Indic voice quality programme (ASR eval sets & WER tracking) | EP-MVP1-19 | ST-MVP1-19-02 | [SCRUM-131](https://rozvaa.atlassian.net/browse/SCRUM-131) (epic [SCRUM-88](https://rozvaa.atlassian.net/browse/SCRUM-88)) | C9 | key format <module>.<area>.<NN_NN> differs from MVP0 proposed keys — M1-MM-09 | OFF: hidden / 404 FEATURE_DISABLED; data readable, no writes lost (story) |

### MVP1 — Resilience (DRAFT)

| Row ID | Item | Detail | Epic ID | Story ID | Jira key | Constraints | Mismatch / note |
|---|---|---|---|---|---|---|---|
| M1-RES-09-02 | cb.llm_api | timeout 10 s external; breaker 50%/20 calls, half-open 30 s; bulkhead 10/tenant; fallback: degrade to manual / template text, never block the core flow | EP-MVP1-09 | ST-MVP1-09-02 | [SCRUM-119](https://rozvaa.atlassian.net/browse/SCRUM-119) (epic [SCRUM-78](https://rozvaa.atlassian.net/browse/SCRUM-78)) | C6, C12 | story |
| M1-RES-19-01 | cb.whatsapp_cloud_api | timeout 10 s external; breaker 50%/20 calls, half-open 30 s; bulkhead 10/tenant; fallback: degrade to manual / template text, never block the core flow | EP-MVP1-19 | ST-MVP1-19-01 | [SCRUM-124](https://rozvaa.atlassian.net/browse/SCRUM-124) (epic [SCRUM-88](https://rozvaa.atlassian.net/browse/SCRUM-88)) | C6, C12 | story |
| M1-RES-19-02 | cb.asr_vendor_api | timeout 10 s external; breaker 50%/20 calls, half-open 30 s; bulkhead 10/tenant; fallback: degrade to manual / template text, never block the core flow | EP-MVP1-19 | ST-MVP1-19-02 | [SCRUM-131](https://rozvaa.atlassian.net/browse/SCRUM-131) (epic [SCRUM-88](https://rozvaa.atlassian.net/browse/SCRUM-88)) | C6, C12 | story |

### MVP1 — Observability (DRAFT)

| Row ID | Item | Detail | Epic ID | Story ID | Jira key | Constraints | Mismatch / note |
|---|---|---|---|---|---|---|---|
| M1-OBS-AI | business metrics: bos_narrative_grounding_failures_total, bos_wa_agent_actions_total{action,result} | RED + logs + spans + SLO + alerts + Grafana panel per story (values above) | EP-MVP1-09, EP-MVP1-19 | ST-MVP1-09-02, ST-MVP1-19-01, ST-MVP1-19-02 | [SCRUM-119](https://rozvaa.atlassian.net/browse/SCRUM-119), [SCRUM-124](https://rozvaa.atlassian.net/browse/SCRUM-124), [SCRUM-131](https://rozvaa.atlassian.net/browse/SCRUM-131) | C12 | story |

### MVP1 — Infra (DRAFT)

| Row ID | Item | Detail | Epic ID | Story ID | Jira key | Constraints | Mismatch / note |
|---|---|---|---|---|---|---|---|
| M1-INF-19-02 | Indic ASR eval | CI eval harness; encrypted audio with retention. | EP-MVP1-19 | ST-MVP1-19-02 | [SCRUM-131](https://rozvaa.atlassian.net/browse/SCRUM-131) (epic [SCRUM-88](https://rozvaa.atlassian.net/browse/SCRUM-88)) | C8, C12 | no API contract; IaC via reviewed PR only |
