# AGENTS.md — Business OS onboarding for AI team members

You are a team member on Business OS. Read these in order before touching code. They are binding.

1. **Big picture** — [product-context.md](standards/product-context.md): what we build, for whom, why.
2. **Architecture** — [architecture.md](standards/architecture.md), [api.md](standards/api.md).
3. **Design** — [ui-ux.md](standards/ui-ux.md).
4. **Coding guardrails** — [coding.md](standards/coding.md), [testing.md](standards/testing.md), [reliability-observability.md](standards/reliability-observability.md), [feature-flags.md](standards/feature-flags.md), [security-privacy.md](standards/security-privacy.md), [definition-of-done.md](standards/definition-of-done.md).
5. **Workflow** — [story-workflow.md](standards/story-workflow.md): how to pick, build, and close a Jira story.

## Hard guardrails (never)
- Commit secrets, keys, tokens, `.env` files, or real customer data.
- Access production, or provision any infrastructure.
- Force-push, rewrite shared history, or push to `main` directly.
- Pick an unassigned story, or edit another service/MFE without a linked story.
- Edit `AGENTS.md` or `standards/**` without Ranjith's approval.
- Introduce a monolith, shared database across services, or a known tech-debt shortcut.

If blocked or unsure: comment on the Jira story and stop.
