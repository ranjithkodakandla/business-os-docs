# ADR-002 Data platform, HA, PITR and pooling
- **Context:** payroll and GST data can't be lost; 11 services on Cloud Run can open too many connections.
- **Options:** Cloud SQL Enterprise; Cloud SQL Enterprise Plus; AlloyDB; CloudNativePG on GKE.
- **₹ costs:** db-g1-small $0.035/h (≈ ₹2,430/month); HA ≈ 2×; a read replica costs about as much as the instance.
- **Decision:** Cloud SQL Postgres 17 in asia-south1, one shared instance with one schema + DB user per service, PgBouncer in transaction mode.
  - MVP0: zonal + PITR (7 days).
  - MVP1: regional HA from the first paying tenant, plus one read replica when reads justify it.
  - MVP3: Enterprise Plus (PITR up to 35 days) + a DR replica in asia-south2.
  - RPO/RTO targets as in §5.
- **Consequences:**
  - PITR must be declared in OpenTofu.
  - Quarterly restore drills.
  - A connection budget is enforced in CI.
  - A service can move to its own instance later without code changes (only its DSN changes).
- **Sources:** https://cloud.google.com/sql/pricing ; https://cloud.google.com/sql/docs/postgres/editions-intro ; https://docs.cloud.google.com/sql/docs/postgres/backup-recovery/configure-pitr



## Related Jira epics

- [SCRUM-84](https://rozvaa.atlassian.net/browse/SCRUM-84)


_Source: [Architecture v1 Google Doc](https://docs.google.com/document/d/1sKoivvz3JLkKPI1Jta0i6XWnWuW57-rSwWICmm1w9VM/edit)_
