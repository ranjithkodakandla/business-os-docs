# Architecture standards

## Shape (non-negotiable)
- [ ] **Microservices**, one per business capability. No monolith, not even a "modular monolith".
- [ ] **Micro frontends**: one shell (top menu bar, auth, i18n, design system) + one independently built/deployed MFE per module.
- [ ] Each service owns its schema. **No cross-service DB reads/writes**; integrate via API or events.

## Service boundaries (initial)
| Service | Owns |
|---|---|
| identity-tenancy | tenants, users, roles, permissions, module flags |
| sales-billing | customers, quotes, orders, GST invoices, credit notes |
| collections-payments | receivables, reminders, UPI links, reconciliation |
| inventory-purchase | items, stock, POs, GRNs, bills, 3-way match |
| notifications | WhatsApp, email, SMS delivery and preferences |
| ai-agents | agent runs, proposals, approvals |
| audit | append-only audit log |
New service = ADR + approval.

## Async with Kafka + outbox
- [ ] State change and its event are written in **one DB transaction** to an `outbox` table; a relay publishes to Kafka.
- [ ] Never publish to Kafka directly inside a request handler.
- [ ] Consumers are **idempotent** (dedupe on event `id`) and tolerate out-of-order delivery.
- [ ] Topics: `<domain>.<entity>.<event>.v<N>`; partition key = `tenant_id` (or aggregate id).
- [ ] Failed messages go to a DLQ with alerting; never silently drop.
- Use sync calls only when the caller needs the answer now; otherwise use events.

## Events: CloudEvents v1.0
- Required: `specversion, id, source, type, time, subject, datacontenttype`, plus extension `tenantid`, `traceparent`.
- `type`: `in.rozvaa.<domain>.<entity>.<event>.v<N>`. Schemas versioned; additive changes only within a version.

## CQRS and data
- [ ] Writes go to the **primary** DB. Reads for lists, search, reports go to **read replicas**.
- [ ] Read-your-own-write paths (e.g. right after save) read from primary or use returned data.
- [ ] Handle replica lag explicitly; document tolerated staleness per endpoint.
- [ ] Postgres. Migrations are forward-only and follow expand → migrate → contract.
- [ ] Redis for cache with TTL and explicit invalidation; cache keys include `tenant_id`.
- [ ] Analytics go to a warehouse via events, never off the OLTP primary.

## Multi-tenancy
- [ ] Default: shared tables with `tenant_id NOT NULL` on every row + Postgres **Row-Level Security**.
- [ ] Tenant context set per transaction (`SET LOCAL app.tenant_id`) from the verified token, never from client input.
- [ ] Every index that serves tenant queries leads with `tenant_id`.
- [ ] Cross-tenant isolation test per table (see testing.md).

## API-first
- [ ] Write/modify the **OpenAPI 3.1** spec before code; review the spec in the PR.
- [ ] URI versioning `/v1/...`; breaking change ⇒ new version + deprecation window.
- [ ] Generate clients/types from the spec; no hand-written duplicates.
- [ ] All mutating endpoints accept `Idempotency-Key` (see api.md).

## Decisions
- Record every significant choice as an ADR in `adr/NNNN-title.md` in this repo (context, decision, consequences).
