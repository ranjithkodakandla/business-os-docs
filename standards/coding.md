# Coding standards

## General
- [ ] Small, single-purpose functions and modules. Names say what, not how.
- [ ] **No dead code, no unused variables/imports/params/exports, no commented-out code, no TODO without a Jira key.**
- [ ] No magic numbers/strings; use named constants or config.
- [ ] Validate at boundaries (HTTP, events, config, third parties); trust typed data inside.
- [ ] Comments explain *why*, not *what*.

## TypeScript / React
- [ ] `strict: true`, `noUnusedLocals`, `noUnusedParameters`, `noImplicitReturns`, `noFallthroughCasesInSwitch`, `exactOptionalPropertyTypes`.
- [ ] No `any`, no non-null `!` without a comment, no `@ts-ignore` (use `@ts-expect-error` + reason).
- [ ] ESLint (typescript-eslint strict, react-hooks, jsx-a11y, import) + Prettier; `knip` for unused files/exports/deps. Zero warnings in CI.
- [ ] Function components + hooks only. Server data via a query library with cancellation; no fetch in render.
- [ ] Types generated from OpenAPI. Runtime-validate external data (zod).
- [ ] Use design-system components from `src/components/ui.tsx`; no ad-hoc colours.

## Python
- [ ] Python 3.12+, FastAPI, Pydantic v2, SQLAlchemy 2 (async) or equivalent.
- [ ] `ruff` (incl. F401/F841 unused, B, S, UP, SIM, ARG), `ruff format`, `mypy --strict`, `vulture` for dead code. Zero findings in CI.
- [ ] Type hints everywhere; no bare `except:`; no mutable default args.

## Errors and exception handling
- [ ] Typed domain errors (`DomainError` subclasses / TS discriminated `Result` or error classes) mapped once to RFC 9457 at the edge.
- [ ] Never swallow exceptions. Catch only what you can handle; otherwise log with context and re-raise.
- [ ] No PII or secrets in error messages or logs.
- [ ] Every external call has a timeout (see reliability-observability.md).

## Memory and resources
- [ ] Close every resource (`with`/`async with`, `try/finally`). Use pooled DB/HTTP clients; never create per request.
- [ ] React: clean up every effect (listeners, intervals, subscriptions, AbortController).
- [ ] No unbounded caches/arrays/maps; set size or TTL. Stream large exports.
- [ ] No module-level mutable state holding request data.

## Performance
- [ ] No N+1 queries; check `EXPLAIN` for new queries; paginate everything.
- [ ] Frontend: route-level code splitting; MFE bundle budget ≤ 200 KB gzip initial.

## Dependencies
- [ ] Add a dependency only if it saves real work; prefer well-maintained, permissive licence (no GPL/AGPL in services).
- [ ] Pin via lockfile; Renovate/Dependabot on; no known high/critical CVEs (`npm audit`, `pip-audit`).
- [ ] Remove unused deps in the same PR.
