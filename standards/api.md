# API standards

## REST
- Base path `/v1`. Plural nouns, kebab-case: `/v1/sales-invoices/{id}`.
- JSON fields snake_case. Timestamps ISO-8601 UTC. Money as `{ "amount": "1234.50", "currency": "INR" }` (string decimals, never floats).
- Methods: GET read, POST create/action, PATCH partial update, PUT full replace, DELETE soft-delete.
- Actions as sub-resources: `POST /v1/invoices/{id}/send`.
- Status codes: 200, 201 (+Location), 202 async, 204, 400, 401, 403, 404, 409, 412, 422, 429, 5xx.
- Optimistic concurrency: ETag + If-Match on updates → 412 on mismatch.

## Errors: RFC 9457 (application/problem+json)
```json
{ "type": "https://errors.rozvaa.in/validation", "title": "Validation failed", "status": 422,
  "detail": "GSTIN is invalid", "instance": "/v1/customers", "code": "CUSTOMER_GSTIN_INVALID",
  "trace_id": "…", "errors": [{ "field": "gstin", "code": "pattern", "message": "…" }] }
```
Stable machine code; human text translatable. Never leak stack traces, SQL, or PII.

## Pagination, filtering, sorting
- Cursor pagination: `?limit=50&cursor=…` → `{ "data": [...], "next_cursor": "…" }`. Max limit 200.
- Filters `?status=paid&created_after=…`; sort `?sort=-created_at`. Whitelist sortable fields.

## Idempotency
- All POST/PATCH accept `Idempotency-Key` (UUID). Required for money, invoice, payment, and message-send endpoints.
- Store key + request hash + response for 24 h per tenant. Same key + same body → replay response; different body → 409.

## Read-your-writes consistency (`X-BOS-Consistency`)
After any write, the service returns the commit LSN in the response header:
```
X-BOS-Consistency: <pg_current_wal_lsn_value>
```
The `@bos/sdk` **must** implement this protocol:
- Store the latest `X-BOS-Consistency` token received per service in memory (session scope).
- On subsequent GET requests to the same service, send the token as `X-BOS-Read-After: <token>`.
- The service checks `pg_last_wal_replay_lsn()` on the replica; if the replica hasn't caught up, the read falls back to the primary.
- Simpler fallback: route all reads to the primary for 5 s after any write (configurable per service).

This is **required in `@bos/sdk`** from MVP0. Without it, MFEs show stale data immediately after a write (e.g. approval queue not updating after approve action), which breaks the agentic confirm-then-act flow.

## AuthN / AuthZ
- Bearer JWT from the identity provider; validate signature, exp, aud, iss at the gateway and in the service.
- `tenant_id` comes from the token only.
- Every endpoint declares required permission in the OpenAPI spec (`x-permission: invoices.write`); enforce server-side, deny by default.
- Object-level checks (BOLA): verify the resource belongs to the tenant and the user's scope.
- Module flag off ⇒ 404/403 consistently, never partial data.

## Agent-initiated calls
- All calls from the ai-agents service carry `X-Agent-Action-ID: <proposal_id>` linking the API call to the audit log entry and the approval record.
- This creates a causal chain: owner instruction → AI proposal → approval → API call → audit record. Without this header the audit log is post-hoc reconstruction, not a causal trace.
- Services must log `agent_action_id` alongside the standard audit fields for any write triggered by an agent.

## Other
- Rate limits per tenant and per user; return 429 with Retry-After.
- Propagate `traceparent`; return `X-Request-Id`.
- Deprecation via `Deprecation` and `Sunset` headers.
