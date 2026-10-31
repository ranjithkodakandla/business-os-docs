# ADR-008 Integration adapters inside owning services
- **Context:** about 10 external providers; the prototype has a useful real/simulated switch.
- **Options:** one shared "integrations" service; adapters inside each owning service; iPaaS (n8n).
- **₹ costs:** ₹0 extra; per-call costs as in §10.
- **Decision:**
  - Each adapter lives in the service that owns the business fact (owner mapping in §10).
  - Adapters use a common adapter SDK with real and simulated implementations, outbox-driven calls, webhooks via the gateway into an inbox table, and credentials in Secret Manager/KMS.
  - n8n is for internal ops only.
- **Consequences:**
  - Switching GSP or BSP is a config change.
  - Simulated adapters double as CI test doubles.
  - A vendor outage is isolated to one service.



## Related Jira epics

- [SCRUM-18](https://rozvaa.atlassian.net/browse/SCRUM-18)
- [SCRUM-72](https://rozvaa.atlassian.net/browse/SCRUM-72)
- [SCRUM-76](https://rozvaa.atlassian.net/browse/SCRUM-76)
- [SCRUM-17](https://rozvaa.atlassian.net/browse/SCRUM-17)


_Source: [Architecture v1 Google Doc](https://docs.google.com/document/d/1sKoivvz3JLkKPI1Jta0i6XWnWuW57-rSwWICmm1w9VM/edit)_
