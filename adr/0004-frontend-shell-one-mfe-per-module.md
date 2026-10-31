# ADR-004 Frontend: shell + one MFE per module (rewritten)
- **Context:** the plan Sheet already lists 9 MFEs (task A1). Module teams need to ship independently.
- **Options:**
  - (a) Module Federation (Vite federation plugin or Module Federation 2.0 / Rspack) with a runtime remote manifest
  - (b) single-spa
  - (c) build-time packages in one app (rejected: one deploy unit)
- **₹ costs:** static hosting on Cloud Storage + Cloud CDN, roughly ₹0–500/month at MVP0/MVP1 (estimate).
- **Decision:** (a). The shell owns routing, auth, tenant, RBAC guard, flags SDK and the AI chat dock. Each module is an MFE with its own build and deploy.
  - React, the router, `@bos/ui` (design system) and `@bos/sdk` (generated API clients, auth, telemetry, flags) are shared singletons with semver ranges.
  - MFEs never import each other; they talk via URL, the shell event bus, or APIs.
  - Each MFE has an error boundary and a flag.
- **Consequences:**
  - Shared-dependency version policy and a CI check for singleton mismatches.
  - Visual regression tests on `@bos/ui`.
  - An MFE outage degrades only its module.
- **Sources:** https://module-federation.io ; plan Sheet MVP0 A1, D1



## Related Jira epics

- [SCRUM-11](https://rozvaa.atlassian.net/browse/SCRUM-11)


_Source: [Architecture v1 Google Doc](https://docs.google.com/document/d/1sKoivvz3JLkKPI1Jta0i6XWnWuW57-rSwWICmm1w9VM/edit)_
