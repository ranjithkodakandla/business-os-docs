# ADR-007 Observability and CI/CD per service
- **Context:** the Sheet contradicts itself (task I3 says Sentry + PostHog; Stack choices says LGTM); there's no CI; 11 services plus 9 MFEs need consistent pipelines.
- **Options:** Sentry + PostHog + Cloud Logging; OTel + Grafana LGTM; Datadog.
- **₹ costs:**
  - Grafana Cloud free tier, then Pro $19 + usage; plan Sheet estimate ₹5,113/month at ~100 tenants.
  - GitHub Actions free for 2,000 min, then $0.006/min.
- **Decision:**
  - OTel + LGTM, with per-service dashboards and SLOs generated from the template.
  - PostHog for analytics only; Sentry dropped.
  - Per-service GitHub Actions reusable workflows with the gates in §8; per-service OpenTofu module; independent deploys with canary.
- **Consequences:**
  - Reword task I3.
  - Build minutes scale with the number of services, so path filters and caching are mandatory.
- **Sources:** https://grafana.com/pricing/ ; https://posthog.com/pricing ; https://github.com/pricing



## Related Jira epics

- [SCRUM-10](https://rozvaa.atlassian.net/browse/SCRUM-10)
- [SCRUM-82](https://rozvaa.atlassian.net/browse/SCRUM-82)


_Source: [Architecture v1 Google Doc](https://docs.google.com/document/d/1sKoivvz3JLkKPI1Jta0i6XWnWuW57-rSwWICmm1w9VM/edit)_
