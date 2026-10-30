# Testing standards

Every story lists concrete test cases per layer. CI blocks merge on any failure.

| Layer | Tool | Rule |
|---|---|---|
| Unit | pytest (Python), Vitest + Testing Library (TS) | ≥ 85% line / 80% branch on changed code; test behaviour, not implementation |
| Integration | pytest + Testcontainers (Postgres, Kafka, Redis) | Real DB with RLS on; outbox → Kafka path covered |
| Contract | Pact (consumer-driven) + OpenAPI schema validation | Every API consumer and event consumer has a pact; provider verification in CI |
| E2E / functional | Playwright | Happy path + key failure per story; EN/HI/TE smoke; mobile viewport 375 px |
| Mutation | mutmut (Python), Stryker (TS) | Required on critical logic (GST/tax, money, rounding, permissions, idempotency): score ≥ 80% |
| Performance | k6 | Meets the story's p95/p99 and throughput budget; no regression > 10% |
| Chaos | Toxiproxy / Chaos Mesh in test env | See below |
| Security | see security-privacy.md | SAST, DAST, dependency scan |
| Accessibility | axe-core in Playwright | Zero serious/critical violations |

## Mandatory cases
- [ ] Tenant isolation: tenant A can never read/write tenant B (API and DB/RLS).
- [ ] Authz: each role allowed/denied as specified.
- [ ] Idempotency: duplicate request and duplicate event produce one effect.
- [ ] Feature flag on **and** off paths; flag off leaves related data and fields intact.
- [ ] Validation edge cases: empty, max length, unicode (Hindi/Telugu), boundary amounts, rounding.

## Chaos tests (per service, before GA)
Inject: dependency down, latency +2 s, packet loss, Kafka lag/broker loss, DB failover to replica/primary switch.
Verify all of:
- [ ] Expected metrics are emitted (error rate, latency, breaker state).
- [ ] Alerts fire within the defined window and an incident is opened.
- [ ] Circuit breakers open and fallbacks/automatic cut-offs engage.
- [ ] No data loss or duplication; system recovers without manual steps.

## Regression gates
- [ ] Full unit + integration + contract suites on every PR; E2E smoke on every PR, full E2E nightly.
- [ ] Bug fixes start with a failing regression test.
- [ ] No skipped/flaky tests merged; quarantine requires a Jira key.
- [ ] Tests are deterministic: no real network, fixed clocks and seeds.
