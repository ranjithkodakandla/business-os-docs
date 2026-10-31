# ADR-002 Data platform, HA, PITR and pooling
- **Context:** payroll and GST data can't be lost; 11 services on Cloud Run can open too many connections.
- **Options:** Cloud SQL Enterprise; Cloud SQL Enterprise Plus; AlloyDB; CloudNativePG on GKE.
- **₹ costs (asia-south1):** db-f1-micro $0.0126/h ≈ ₹875/month; db-g1-small $0.042/h ≈ ₹2,917; HA = 2×; a replica costs the same as an instance. 11 instances ≈ ₹13.6k/month at MVP0 and ≈ ₹32.7k at MVP1 (overview §6c).
- **Decision:** Cloud SQL Postgres 17 in asia-south1, **one instance per service** (database per service, per AGENTS.md; no shared instance).
  - HA, PITR, backups, replicas and pooling are all per instance.
  - MVP0: all instances zonal, db-f1-micro by default, automated backups + PITR (7 days) on every instance; no HA. HA at MVP0 only if a money/identity service takes real money from pilots (justify in the PR).
  - MVP1: regional HA on identity-tenancy, billing-invoicing and collections-payments from the first paying tenant; others stay zonal. One read replica on billing-invoicing; others only when their own metrics justify it.
  - Pooling: in-process pool per service sized to its own instance's `max_connections`; PgBouncer sidecar only for a service that needs it.
  - MVP3: Enterprise Plus (PITR up to 35 days) + a DR replica in asia-south2.
  - RPO/RTO targets as in §5.
- **Consequences:**
  - PITR must be declared in OpenTofu.
  - Quarterly restore drills.
  - A per-service connection budget is enforced in CI.
  - 11× the instances to patch, monitor and drill: OpenTofu module per instance, restore drill rotates one instance per quarter.
  - Cloud SQL Auth/IAM DB auth per service account; no service holds another's credentials.
- **Sources:** https://cloud.google.com/sql/pricing ; https://cloud.google.com/sql/docs/postgres/editions-intro ; https://docs.cloud.google.com/sql/docs/postgres/backup-recovery/configure-pitr



## Related Jira epics

- [SCRUM-84](https://rozvaa.atlassian.net/browse/SCRUM-84)


_Source: [Architecture v1 Google Doc](https://docs.google.com/document/d/1sKoivvz3JLkKPI1Jta0i6XWnWuW57-rSwWICmm1w9VM/edit)_
