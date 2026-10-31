# ADR-009 CQRS-style data access, caching and analytics
- **Context:** reads (lists, reports, AI context) far outnumber writes; the primary must stay small and cheap; 10k-tenant reporting must not hit OLTP.
- **Options:**
  - all reads on the primary
  - read replicas with consistency tokens
  - full CQRS with separate read databases per service
  - analytics in BigQuery vs ClickHouse
- **₹ costs:**
  - a replica is per instance and costs the same as its primary (g1-small ≈ ₹2,917/month in asia-south1); MVP1 adds one, on billing-invoicing only
  - BigQuery free tier first (amounts not re-checked)
  - ClickHouse Cloud/self-host not priced here (**Unverified**)
- **Decision:**
  - Writes → the service's own primary; tolerant reads → that service's own replica where one exists (replicas per instance, none at MVP0), with LSN read-your-writes tokens (5 s primary-stickiness fallback) and lag-based eviction (§6a).
  - Redis/Valkey cache-aside with tenant-scoped keys, invalidated by Kafka events.
  - Analytics and any cross-service reporting: BigQuery fed from Kafka (every service's events), never queries across instances.
  - Per-service read models (projections) built from events only where a screen needs cross-domain data.
- **Consequences:**
  - The data layer has a read/write router from W3, even before a replica exists.
  - Staleness rules are documented per endpoint in OpenAPI (`x-consistency: strong | eventual`).
- **Sources:** https://cloud.google.com/sql/docs/postgres/replication ; https://cloud.google.com/bigquery/pricing ; https://www.postgresql.org/docs/current/functions-admin.html



## Related Jira epics

- [SCRUM-78](https://rozvaa.atlassian.net/browse/SCRUM-78)


_Source: [Architecture v1 Google Doc](https://docs.google.com/document/d/1sKoivvz3JLkKPI1Jta0i6XWnWuW57-rSwWICmm1w9VM/edit)_
