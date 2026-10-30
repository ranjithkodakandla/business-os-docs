# API standards

## REST
- [ ] Base path `/v1`. Plural nouns, kebab-case: `/v1/sales-invoices/{id}`.
- [ ] JSON fields `snake_case`. Timestamps ISO-8601 UTC. Money as `{ "amount": "1234.50", "currency": "INR" }` (string decimals, never floats).
- [ ] Methods: GET read, POST create/action, PATCH partial update, PUT full replace, DELETE soft-delete.
- [ ] Actions as sub-resources: `POST /v1/invoices/{id}/send`.
- [ ] Status codes: 200, 201 (+`Location`), 202 async, 204, 400, 401, 403, 404, 409, 412, 422, 429, 5xx.
- [ ] Optimistic concurrency: `ETag` + `If-Match` on updates → 412 on mismatch.

## Errors: RFC 9457 (`application/problem+json`)
```json
{ "type": "https://errors.rozvaa.in/validation", "title": "Validation failed", "status": 422,
  "detail": "GSTIN is invalid", "instance": "/v1/customers", "code": "CUSTOMER_GSTIN_INVALID",
  "trace_id": "…", "errors": [{ "field": "gstin", "code": "pattern", "message": "…" }] }
```
- [ ] Stable machine `code`; human text translatable. Never leak stack traces, SQL, or PII.

## Pagination, filtering, sorting
- [ ] Cursor pagination: `?limit=50&cursor=…` → `{ "data": [...], "next_cursor": "…" }`. Max `limit` 200.
- [ ] Filters `?status=paid&created_after=…`; sort `?sort=-created_at`. Whitelist sortable fields.

## Idempotency
- [ ] All POST/PATCH accept `Idempotency-Key` (UUID). Required for money, invoice, payment, and message-send endpoints.
- [ ] Store key + request hash + response for 24 h per tenant. Same key + same body → replay response; different body → 409.

## AuthN / AuthZ
- [ ] Bearer JWT from the identity provider; validate signature, `exp`, `aud`, `iss` at the gateway and in the service.
- [ ] `tenant_id` comes from the token only.
- [ ] Every endpoint declares required permission in the OpenAPI spec (`x-permission: invoices.write`); enforce server-side, deny by default.
- [ ] Object-level checks (BOLA): verify the resource belongs to the tenant and the user's scope.
- [ ] Module flag off ⇒ 404/403 consistently, never partial data.

## Other
- [ ] Rate limits per tenant and per user; return 429 with `Retry-After`.
- [ ] Propagate `traceparent`; return `X-Request-Id`.
- [ ] Deprecation via `Deprecation` and `Sunset` headers.
