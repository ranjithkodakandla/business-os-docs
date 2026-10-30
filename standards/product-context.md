# Product context

## Vision
An **AI-native Business OS for Indian MSMEs**: one place to sell, bill (GST), collect, buy, stock, and run people — with AI agents doing the routine work and humans approving.

## Who it's for
- Owners of Indian micro, small and medium businesses: manufacturing, trading, services.
- Their accountants/billing staff, sales, store and operations teams.
- Users who are busy, mobile-first, often non-technical, and work in English, Hindi or Telugu.

## Why we build it
- MSMEs juggle Tally, spreadsheets, WhatsApp and portals; cash is stuck in receivables.
- Incumbents (Tally, Zoho, Odoo, TranZact) are form-first. We are agent-first: the system proposes, the user approves.
- Reuse proven patterns from those products (cite the source); innovate only where we differ.

## Wedge: collections
Get MSMEs paid faster: invoices → reminders (WhatsApp/email) → UPI links → reconciliation → follow-up by AI. Every early feature must strengthen this loop.

## Scope
- **MVP0** — foundations: tenancy, identity/roles, module flags, GST invoicing, customers, collections loop, outbox/events, audit, observability.
- **MVP1** — expand: payments & reconciliation, inventory & purchase (PO-GRN-bill), notifications, AI agents on top. **Payroll is out of MVP1.**
- Source of truth for scope: Jira project `SCRUM` epics on rozvaa.atlassian.net.

## Why the architecture looks the way it does (Keka lessons)
- **Scale:** monoliths and shared DBs hit walls; → microservices, micro frontends, read replicas, Kafka.
- **Observability:** incidents were hard to see and diagnose; → metrics, traces, SLOs and alerts in every story.
- **Cost:** pay for what scales; → stateless services, caching, replicas for reads, per-module flags.
- **No tech debt:** shortcuts compound. No known debt is merged; if forced, it needs a Jira ticket and approval.
