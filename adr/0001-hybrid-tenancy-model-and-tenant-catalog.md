# ADR-001 Hybrid tenancy model and tenant catalog
- **Context:** 10k+ tenants, mostly small MSMEs, a few needing isolation; MVP0 exit criterion is zero tenant leaks; every service owns its own tables.
- **Options:** (a) shared tables + RLS only; (b) schema per tenant for everyone; (c) a DB per tenant for everyone; (d) hybrid tiers routed through a catalog.
- **₹ costs:**
  - (a) is the cheapest.
  - (c) is about $150/tenant/month per dedicated DB (plan Sheet estimate, **Unverified**), impossible at ₹4,999 plans.
  - (d) costs the same as (a) for most tenants, plus (c) only where it is paid for.
- **Decision:** (d), with the catalog owned by **identity-tenancy**.
  - T1: each service's tables on the shared instance with RLS.
  - T2: per-tenant schemas within each service's database area (paid add-on).
  - T3: a dedicated Cloud SQL instance hosting all service schemas for that tenant (Enterprise).
  - The gateway resolves the tier from the catalog (Redis-cached) and passes a signed tenant context; each service's data layer picks the pool.
- **Consequences:**
  - Every service's migrations run per tier through a shared migration runner.
  - Promotion tooling is needed by MVP2/MVP3.
  - Cross-tenant analytics happen only in BigQuery.
- **Promotion triggers:** as in §4.
- **Sources:** plan Sheet Architecture & Flags and MVP3 I4; https://www.postgresql.org/docs/current/ddl-rowsecurity.html



## Related Jira epics

- [SCRUM-5](https://rozvaa.atlassian.net/browse/SCRUM-5)
- [SCRUM-6](https://rozvaa.atlassian.net/browse/SCRUM-6)


_Source: [Architecture v1 Google Doc](https://docs.google.com/document/d/1sKoivvz3JLkKPI1Jta0i6XWnWuW57-rSwWICmm1w9VM/edit)_
