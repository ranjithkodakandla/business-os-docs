# Definition of Done

A story is done only when every applicable box is checked and evidence is on the Jira story.

## Scope
- [ ] All acceptance criteria met exactly; nothing out of scope added.
- [ ] OpenAPI / event schema updated and reviewed before code; APIs merged before their UI.

## Code
- [ ] Follows coding.md; lint, type check, dead-code check (knip/vulture) pass with zero warnings.
- [ ] No unused vars/imports/deps, no commented-out code, no TODO without Jira key.
- [ ] Typed errors mapped to RFC 9457; no swallowed exceptions; timeouts on all calls.
- [ ] No memory/resource leaks (effects cleaned up, resources closed, bounded caches).

## Architecture
- [ ] Stays inside its own service/MFE; no cross-service DB access.
- [ ] Writes on primary, reads on replica where applicable; events via outbox in CloudEvents format.
- [ ] Tenant isolation via `tenant_id` + RLS; idempotency on mutations and consumers.

## Tests (testing.md)
- [ ] Unit, integration, contract, E2E written and green; coverage thresholds met.
- [ ] Mutation score ≥ 80% on critical logic.
- [ ] Performance budget met (k6). Chaos cases pass where the story adds a dependency.
- [ ] Tenant isolation, authz, idempotency, flag on/off tests included.
- [ ] No regressions: full suite green.

## Reliability and observability
- [ ] RED + business metrics, structured logs, traces emitted.
- [ ] SLOs and alerts defined; runbook linked; Grafana panel added.
- [ ] Circuit breaker, retries and fallback on every external/inter-service call.

## Feature flag
- [ ] Behind a flag; default off; kill switch works ≤ 10 s.
- [ ] Flag off breaks no related fields/data; backward compatible (expand → contract).

## Security and privacy
- [ ] OWASP checks done; SAST, SCA, secret scan clean.
- [ ] PII/financial fields encrypted and masked; nothing sensitive in logs.
- [ ] Audit log entries for writes and sensitive reads; DPDP/GDPR impacts noted.
- [ ] No secrets committed.

## UI (if applicable)
- [ ] Uses design-system tokens/components only.
- [ ] Empty, loading, error, offline states done.
- [ ] WCAG 2.2 AA (axe clean, keyboard-only pass); EN/HI/TE strings; mobile 360–768 px pass.

## Delivery
- [ ] PR reviewed and approved; CI green; Conventional Commit messages.
- [ ] Docs/ADR/runbook updated where behaviour changed.
- [ ] Jira updated: status, PR link, test evidence (CI run, screenshots), flag name.
