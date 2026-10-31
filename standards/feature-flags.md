# Feature flags

## Rules
- Every story ships behind a flag: `<module>.<feature>` (e.g. `collections.whatsapp_reminders`).
- Flags are evaluated server-side per tenant/user; UI reads the same evaluated result.
- Default off for new features; tests cover on and off.
- Each flag has an owner, Jira key, and removal date. Remove flag and dead branch within 2 sprints after 100% GA.

## Rollout stages
1. **Internal / alpha** — staff tenants only.
2. **Beta** — opted-in tenants.
3. **Gradual** — 5% → 25% → 50% → 100% by tenant, gated on SLOs and error budget.
4. **GA** — flag removed within 30 days.
5. **Kill switch** — any flag can be forced off globally or per tenant; takes effect in ≤ 10 s, no deploy.

## Graduation gate for agent-facing flags
A flag that gates an AI agent capability **must not graduate from Beta → Gradual** unless:
- The eval harness passes ≥ 85% on the 200-prompt eval set for that capability.
- LLM error rate for that intent is < 2% over the last 7 days on beta tenants.
- No P1/P2 guardrail-related incidents in the last 7 days.

This is a hard gate enforced in CI: the gradual rollout job reads the latest eval result from the eval artefact store and blocks if the threshold is not met.

## Turning a flag off must not break anything
- Data written while on stays valid and readable while off. Never delete/null related fields on flag-off.
- Related fields keep their values; UI hides the feature's controls but shows existing data read-only where needed.
- APIs remain backward compatible: off ⇒ endpoint returns a consistent 403/404, other endpoints unchanged.
- Event consumers ignore events they cannot handle; producers keep schemas stable.
- Module flag off ⇒ no orphaned jobs, reminders, or webhooks keep running.

## Backward compatibility: expand → migrate → contract
- **Expand:** add new columns/fields/endpoints as optional; old code keeps working.
- **Migrate:** dual-write / backfill; switch reads behind a flag.
- **Contract:** remove old fields only after all consumers moved (pact-verified) and one release has passed.
- Never rename or remove a field, column, or event attribute in a single release.
