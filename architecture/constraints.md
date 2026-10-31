# Key constraints for MVP0/MVP1 builders

Binding for every story. Updated 31 Oct 2026: database per service (Ranjith's decision).

1. Every tenant-owned table has a non-null tenant_id and an RLS policy; each service connects as its own non-owner DB user and sets app.tenant_id per transaction. No exceptions without an allowlisted "global" table.
2. tenant_id comes only from the verified auth token (via the gateway-signed tenant context), never from a URL, header or request body set by the client.
3. Database per service: each service owns its own Cloud SQL instance (bos-<service>-pg) and database; no shared instance, no cross-instance queries, no service holds another's credentials. All DB access goes through the platform data layer, which resolves (tenant, service) via the tenant catalog and routes writes to that service's primary and tolerant reads to its own replica if it has one; no hard-coded connection strings.
4. Each service owns its data and calls other services only via their APIs or events; no shared tables, no cross-service joins, no reading another service's schema.
5. Every API is versioned under /v1 and defined in OpenAPI first in the contracts repo; every event has a registered schema; breaking changes need a new version.
6. Async first: any side effect or cross-service update that does not need an immediate answer is an event, written to the service's outbox in the same transaction and relayed to Kafka; consumers are idempotent.
7. External systems are called only through an adapter (real and simulated) inside the owning service; no vendor SDK in domain code.
8. Secrets live only in Secret Manager (per-tenant credentials KMS-encrypted); nothing in git, images, .env in prod, or JSON on disk; gitleaks blocks the PR.
9. Every MFE and every new user-facing capability ships behind a feature flag with a kill switch; plan entitlements are checked separately from rollout flags.
10. Schema changes are forward-only expand/contract migrations that run unchanged on shared, schema-per-tenant and dedicated databases.
11. The tenant-leak suite, contract and schema-compatibility checks, migration test, connection-budget check and secret scan must pass before merge; nobody bypasses the gate.
12. Instrument with OpenTelemetry only (traces across gateway, services and Kafka headers, logs, metrics, errors) tagged with tenant_id, service and request_id; no Sentry; PostHog only for product analytics; no PII in logs.
13. AI agents act with the invoking user's permissions and only through other services' APIs, never move money, file returns or send above the approval limit without a human approval, and every AI write is audited.
14. Cache keys are always tenant-prefixed and owned by one service, and are invalidated by that service's events; anything deciding money, stock or permissions reads from the primary.
15. All data, backups, logs and processing stay in India regions (asia-south1 primary, asia-south2 DR); PITR is on for every database; LLM calls go only through LiteLLM with per-tenant budgets; money/GST amounts are stored as integer paise.
16. PII and financial identifiers (PAN, bank account, salary, contact details of data principals) are stored only as field-level ciphertext under the tenant/subject DEK, searched only via blind indexes, masked by default in UI, APIs and logs; unmasking needs a permission and is audited.
17. Never store a full Aadhaar number in any business database; use a masked value or an Aadhaar Data Vault reference key only.
18. Kafka events, caches, analytics tables and logs never carry raw PII: IDs, pseudonyms or field ciphertext only, and every PII field in a schema is tagged x-pii.
19. Every processing of personal data cites a recorded consent or another lawful purpose from the purposes catalogue, and honours withdrawal, retention holds and erasure events (crypto-shred the subject/tenant key).
20. No standing human access to production: per-service least-privilege service accounts, JIT elevation, break-glass only with alerting; every access to PII, keys and admin actions lands in the hash-chained, WORM-locked audit trail.
21. HA, PITR (7 days), backups, CMEK and replicas are configured per instance in OpenTofu; HA only on identity-tenancy, billing-invoicing and collections-payments until metrics justify more; replicas only where that service's read metrics justify them.
22. Each service's connection budget (max-instances × pool max + headroom) must fit its own instance's max_connections; CI fails the deploy otherwise.
23. Cross-service reporting and analytics use BigQuery fed from Kafka or event-built read models, never direct reads of another service's instance.
