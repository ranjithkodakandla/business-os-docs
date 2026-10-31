# Disaster recovery runbook

**Incident commander (IC):** the founder (Ranjith) until an on-call rotation exists.
**Scope:** database per service: 11 Cloud SQL instances (identity-tenancy, billing-invoicing, collections-payments, inventory, purchase, sales-crm, notifications, documents, ai-agents, feature-flags, audit) plus the tenant catalog. Every DB step below targets **only the affected service's instance**.
See [architecture overview §5](../../architecture/overview.md#5-ha-dr-and-pitr) for RPO/RTO by phase.

## First 5 minutes (all scenarios)
1. Declare the incident, open an entry in the ISMS incident log, name the IC.
2. Identify the affected service(s) and instance(s) from Grafana (per-service RED + PgBouncer dashboards).
3. Decide whether a kill switch (feature-flags) limits blast radius.

## Scenario 1: Cloud SQL zone failover (regional HA from MVP1)
- **Detect:** PgBouncer `cl_waiting` / server-connection errors for one service's pool; Cloud SQL failover event.
- **Expect:** HA failover completes automatically; the service's `/readyz` must return healthy within **3 min**.
- **Escalate:** not healthy at **5 min** → open a GCP support case at P1, IC informed.
- **Verify:** run the tenant-leak gate (cross-tenant read test) against the service before closing; check the outbox drained and consumer lag recovered.
- MVP0 (zonal, no HA): restore from backup / PITR into a new instance for that service; RTO ≤ 4 h.

## Scenario 2: Logical deletion / corruption (bad migration, bug, operator error)
RPO 5 min, RTO 1 h.
1. Set the affected tenant(s) to `tenant.readonly` (banner shows).
2. Find the last good time from the audit service (who/what/when, `agent_action_id` if agent-triggered).
3. PITR-clone **the affected service's instance only** to a new instance at that timestamp.
4. Export the affected tenant's rows (`WHERE tenant_id = …`) from the clone.
5. Apply to the primary in a **single transaction**, verify row counts/checksums, re-emit needed events via the outbox.
6. Lift `tenant.readonly`; notify the tenant within **1 h** of confirmation.
7. Keep the clone **7 days**, then delete it (logged).

## Scenario 3: Region failure (asia-south1)
**MVP3 Enterprise only.** RPO 15 min, RTO 1 h.
1. IC confirms a regional outage (GCP status + own probes).
2. Promote the asia-south2 cross-region replicas, per service instance and catalog, in order: catalog → identity-tenancy → feature-flags → audit → remaining services.
3. Point the gateway / Cloud Run deployments in asia-south2 at the promoted instances; run the tenant-leak gate.
4. Non-Enterprise tenants: status-page notice; restore from backups when the region returns.

## Scenario 4: Confluent (Kafka) unavailable
- Writes continue: each service's outbox absorbs events (Postgres).
- Alert when `outbox_pending` > **10k** for any service.
- Still down after **2 h** → P1; review outbox table growth and connection budget per instance.
- On recovery: relay drains in order (key = `tenant_id:aggregate_id`); watch consumer lag and DLQs; confirm audit lag back under 10 s.

## Quarterly drill checklist (log each in the ISMS incident log)
- [ ] PITR-clone one service instance (rotate the service each quarter) and time restore-to-ready.
- [ ] Logical-deletion rehearsal on a test tenant, end to end, measured against RTO 1 h.
- [ ] Simulated Confluent outage in staging: confirm outbox absorption and the `outbox_pending` alert.
- [ ] HA failover test on one instance (MVP1+): `/readyz` within 3 min.
- [ ] Region failover (MVP3, semi-annual).
- [ ] Record actual RPO/RTO, gaps, and follow-up tickets.
