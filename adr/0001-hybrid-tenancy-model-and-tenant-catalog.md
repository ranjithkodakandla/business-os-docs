# ADR-001 Hybrid tenancy model and tenant catalog
- **Context:** 10k+ tenants, mostly small MSMEs, a few needing isolation; MVP0 exit criterion is zero tenant leaks; every service owns its own tables.
- **Options:** (a) shared tables + RLS only; (b) schema per tenant for everyone; (c) a DB per tenant for everyone; (d) hybrid tiers routed through a catalog.
- **₹ costs:**
  - (a) is the cheapest.
  - (c) is about $150/tenant/month per dedicated DB (plan Sheet estimate, **Unverified**), impossible at ₹4,999 plans.
  - (d) costs the same as (a) for most tenants, plus (c) only where it is paid for.
- **Decision:** (d), with the catalog owned by **identity-tenancy**.
  - Tiers apply **per service database** (database per service, 11 instances; no shared instance).
  - T1: shared tables with RLS inside each service's own instance.
  - T2: per-tenant schema inside that service's instance (paid add-on), per service.
  - T3: a dedicated instance per service for that tenant (Enterprise); a tenant may be T3 for billing and T1 for notifications.
  - The catalog maps **tenant × service → tier, instance, database, schema, KMS key**. The gateway passes a signed tenant context; each service's data layer looks up only its own (tenant, service) row (Valkey-cached) and picks the pool.
- **Consequences:**
  - Every service's migrations run per tier and per instance through the shared migration-runner library, iterating that service's catalog rows.
  - Promotion is per service (move one service's rows for a tenant), so it can be staged.
  - Promotion tooling is needed by MVP2/MVP3.
  - Cross-tenant analytics happen only in BigQuery.
- **Promotion triggers:** as in §4.
- **Sources:** plan Sheet Architecture & Flags and MVP3 I4; https://www.postgresql.org/docs/current/ddl-rowsecurity.html



## Related Jira epics

- [SCRUM-5](https://rozvaa.atlassian.net/browse/SCRUM-5)
- [SCRUM-6](https://rozvaa.atlassian.net/browse/SCRUM-6)


_Source: [Architecture v1 Google Doc](https://docs.google.com/document/d/1sKoivvz3JLkKPI1Jta0i6XWnWuW57-rSwWICmm1w9VM/edit)_
