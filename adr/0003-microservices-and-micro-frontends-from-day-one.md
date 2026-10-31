# ADR-003 Microservices and micro frontends from day one (rewritten)
- **Context:** Ranjith's core principle is that the team builds in parallel with no known tech debt and no monolith of any kind. The prototype shares `src/lib` between server and client with no boundaries.
- **Options:**
  - (a) modular monolith (rejected by principle)
  - (b) microservices + MFEs with one shared Cloud SQL instance (schema per service)
  - (c) microservices + MFEs with a database (Cloud SQL instance) per service
- **₹ costs:**
  - (b) ≈ ₹6–7.5k/month at MVP0 and ₹15–20k at MVP1 (estimate).
  - (c) ≈ ₹16–18k/month at MVP0 and ₹39.5–44.5k at MVP1: +≈ ₹10.3k and +≈ ₹22.9k for Cloud SQL (overview §6b/§6c).
- **Decision:** (c), Ranjith's decision 31 Oct 2026, in line with AGENTS.md (no shared database across services).
  - 11 domain services (§3), each with its own repo or package, deploy, Cloud SQL instance, DB user/service account, outbox, OpenAPI spec and dashboard.
  - An API gateway in front.
  - Async-first over Kafka.
  - New services are cut from `service-template`, so one more service costs minutes, not bespoke infra.
- **Consequences:**
  - Contract-first discipline is mandatory (contracts repo, mocks, Pact).
  - Distributed tracing is needed from W3.
  - Local dev uses docker compose.
  - Cross-domain screens compose data in the MFE or in a read model built from events, never with cross-service joins.
- **Sources:** https://cloud.google.com/run/pricing ; https://microservices.io/patterns/microservices.html ; plan Sheet Architecture & Flags (Services list)



## Related Jira epics

- [SCRUM-8](https://rozvaa.atlassian.net/browse/SCRUM-8)
- [SCRUM-11](https://rozvaa.atlassian.net/browse/SCRUM-11)


_Source: [Architecture v1 Google Doc](https://docs.google.com/document/d/1sKoivvz3JLkKPI1Jta0i6XWnWuW57-rSwWICmm1w9VM/edit)_
