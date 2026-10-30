# Security and privacy

## Law
- [ ] **DPDP Act 2023** and **GDPR**: lawful purpose, explicit consent where needed, purpose limitation, data minimisation.
- [ ] Data principal rights: access, correction, erasure, grievance — via API, with audit.
- [ ] Retention per data class documented; statutory records (GST invoices: 8 years) retained, others deleted/anonymised on schedule.
- [ ] Breach response runbook; notify per law.
- [ ] **Data residency: India** (all primary data, backups, logs).

## Encryption
- [ ] In transit: **TLS 1.3** (1.2 minimum for legacy partners), HSTS; mTLS between services.
- [ ] At rest: **AES-256** for DBs, backups, object storage, Kafka.
- [ ] **Per-tenant keys** in a cloud KMS (envelope encryption); key rotation yearly and on demand.
- [ ] **Field-level encryption** for PII and financial data: PAN, Aadhaar, bank account/IFSC, UPI ID, phone, email, salary. Use deterministic tokens only where search is required.

## Masking and minimisation
- [ ] Mask in UI by default (e.g. `XXXXXX1234`); reveal requires permission and is audited.
- [ ] Never log PII, secrets, tokens, or full payloads. Redact in logs, traces and error reports.
- [ ] Non-prod uses synthetic data only.

## Access and audit
- [ ] Least privilege for users, services and humans; RBAC from identity-tenancy; deny by default.
- [ ] Append-only audit log: who, what, when, tenant, before/after (masked), source IP, trace id — for every write and every sensitive read.

## Application security
- [ ] OWASP Top 10 and API Security Top 10 addressed per story (authz/BOLA, injection, SSRF, mass assignment).
- [ ] Parameterised queries only; output encoding; CSP; secure cookies; CSRF protection where cookies are used.
- [ ] File uploads: type/size checks, malware scan, stored outside web root.
- [ ] **SAST** (Semgrep/CodeQL), **SCA** (npm audit, pip-audit), **secret scanning** (gitleaks) on every PR; **DAST** (OWASP ZAP) nightly on staging. High/critical = merge blocked.

## Secrets
- [ ] Secrets live in a secret manager only. Never in code, git, `.env` committed, logs, tickets or chat.
- [ ] `.env*` in `.gitignore`; commit `.env.example` with placeholders only.
- [ ] Leaked secret ⇒ rotate immediately and report.
