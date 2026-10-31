# ADR-006 Feature flags and entitlements
- **Context:** rollout stages and a <10 s kill switch are defined in the plan Sheet; the prototype has only a client-side stub.
- **Options:** self-built service; Unleash OSS; Flagsmith; PostHog flags; LaunchDarkly.
- **₹ costs:**
  - self-built: ₹0 (plan Sheet)
  - Unleash OSS: ₹0 + hosting
  - LaunchDarkly: paid (not priced here)
- **Decision:**
  - A **feature-flags microservice** implementing OpenFeature, with sticky tenant hashing, per-tenant overrides, audit, and the kill switch over Kafka + Redis.
  - Swap to Unleash behind the same OpenFeature interface if targeting rules outgrow it.
  - Entitlements are owned by identity-tenancy.
- **Consequences:**
  - The shell and every service evaluate flags locally from a cached snapshot, so flag-service downtime doesn't take features down.
  - Stale flags (>30 days after GA) fail a CI warning.
- **Sources:** plan Sheet Architecture & Flags; https://openfeature.dev ; https://github.com/Unleash/unleash



## Related Jira epics

- [SCRUM-7](https://rozvaa.atlassian.net/browse/SCRUM-7)


_Source: [Architecture v1 Google Doc](https://docs.google.com/document/d/1sKoivvz3JLkKPI1Jta0i6XWnWuW57-rSwWICmm1w9VM/edit)_
