# Story workflow — the operating loop

Jira: **rozvaa.atlassian.net**, project **SCRUM**. Each AI tool is a team member with a label:
`agent:cursor`, `agent:claude`, `agent:copilot`, `agent:codex`, `agent:windsurf` (or set as assignee).

## Loop
1. **Pick** — take only a story assigned to you (assignee or your `agent:*` label), status *To Do*, with no unresolved blockers. Prefer API stories before their UI stories (check "blocks" links).
2. **Start** — move it to *In Progress*. Comment your plan: approach, files/services touched, API/event changes, tests, flag name.
3. **Branch** — from latest `main`: `<type>/SCRUM-<id>-<short-slug>` (e.g. `feat/SCRUM-42-invoice-api`).
4. **Contract first** — update OpenAPI / event schema and pacts; build the API before the UI.
5. **Build** — work **only** within the story's scope and its own service/MFE. Follow the acceptance criteria exactly; do not add extras.
6. **Verify** — satisfy every item in definition-of-done.md; run the full local test suite.
7. **PR** — open a PR titled `SCRUM-<id>: <summary>`; body lists AC → evidence, tests, flag, risks, screenshots for UI.
8. **Update Jira** — status *In Review*, comment with PR link and test evidence (CI run, coverage, mutation score, screenshots).
9. **Close** — after approval and merge, move to *Done*.

## Commits
- Conventional Commits with the Jira key: `feat(collections): add UPI link endpoint [SCRUM-42]`.
- Types: `feat, fix, refactor, test, docs, chore, perf, build, ci`. Small, focused commits; each one builds.

## Parallel-work rules
- [ ] One story ⇔ one branch ⇔ one PR. One service/MFE per story unless the story says otherwise.
- [ ] Integrate only through published contracts (OpenAPI, events, pacts). No cross-service DB access or imports.
- [ ] Need a change in another service? Ask for a linked story; do not edit it yourself.
- [ ] Shared design-system or contract changes are additive only and need their own story.
- [ ] Rebase on `main` before PR; resolve conflicts only in files your story owns.

## Blockers
- Unclear AC, missing contract, failing dependency, or a needed change outside your scope ⇒ comment on the Jira story (what, why, proposed options), set it to *Blocked*, and **stop**. Do not guess.

## Hard guardrails
- Never commit secrets, keys, tokens or real customer data.
- Never access production or provision infrastructure.
- Never force-push or push directly to `main`.
- Never pick unassigned stories or edit others' services without a linked story.
- Never edit `AGENTS.md` or `standards/**` without Ranjith's approval.
