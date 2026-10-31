# Security & compliance

## 6. Security and DPDP

- **Secrets:** all credentials in Secret Manager; per-tenant integration credentials (GSP, BSP, PG keys) encrypted with Cloud KMS envelope keys (one KEK per tier; per-tenant KEK for T3 / from ~100 tenants per Sheet PO18). No secrets in repo, env files in images, or JSON on disk. gitleaks in CI and pre-commit. **Immediate action:** rotate the hardcoded key in `server/gmail-bridge.gs` and scan git history.
- **Auth:** OIDC via Identity Platform (email OTP + phone OTP + Google) now; Keycloak for enterprise SSO/SAML later (ADR on identity follows the first 8). Short-lived JWT with `tenant_id`, `user_id`, `roles`; server never trusts client-sent tenant.
- **RBAC:** role → permission (`module.resource.action`) matrix per tenant, plus approval limits (e.g. agent auto-limit ₹50,000 from the prototype). AI agents act as a named service principal with the *invoking user's* permissions, never more. Every write is audited (who/what/agent, before/after).
- **Residency:** all tenant data, backups, logs and LLM calls in India regions (asia-south1 primary, asia-south2 DR). LLM: prefer providers/regions serving from India; record each provider's processing location in the DPA register (Unverified per provider).
- **DPDP:** consent and notice records per data principal (employees, customers' contacts); purpose tagging on PII columns; retention schedule (e.g. invoices/books 8 years under Companies Act/GST; attendance/payroll per labour codes; marketing contacts deleted on withdrawal); data-principal request workflow (access/erase) with 30-day SLA target; breach runbook. PII masked in logs (OTel processor) and in LLM prompts where not needed.
- **Edge:** Cloud Armor WAF, rate limits per tenant and per API key, TLS only.



Full decision: [ADR-010](../adr/0010-data-security-privacy-and-compliance.md)

## ADR-010 Data security, privacy and compliance

**Context:** tenant data includes GSTINs, PAN, bank accounts, payroll, employees' and customers' personal data, and possibly Aadhaar. A tenant leak or breach would end the business. We must comply with:

| Law or rule | Status | Source |
|---|---|---|
| DPDP Act 2023 + DPDP Rules 2025 | Rules notified 13 Nov 2025. Rules 1, 2 and 17–21 apply now. Rule 4 (Consent Managers) applies from ~13 Nov 2026. Rules 3, 5–16, 22 and 23 (notice, security, breach, retention, rights) apply from ~13 May 2027. We build to the full rules now. | https://www.dpdpa.in/dpdpa_rules_2025/dpdpa_rules_english_2.pdf ; https://dpdpa.in/notification_timeline_act.pdf ; https://www.pib.gov.in/PressReleasePage.aspx?PRID=2190014 |
| GDPR | Applies to EU data subjects (export customers, EU contacts) | https://gdpr-info.eu/art-33-gdpr/ |
| CERT-In Directions (28 Apr 2022) | Report incidents within 6 h; keep ICT logs 180 days in India | https://www.cert-in.org.in/PDF/CERT-In_Directions_70B_28.04.2022.pdf |
| UIDAI Aadhaar Data Vault rules | In force | https://uidai.gov.in/images/Circular-No.14_of-2025.pdf |

**Options:**
- (a) Platform defaults only: Google's default AES-256 at rest and TLS.
- (b) (a) + application envelope encryption with per-tenant data keys, field-level encryption and crypto-shredding.
- (c) A dedicated KMS key per tenant for everyone, or HSM/EKM for everyone.

**Decision: (b), with (c) sold as an Enterprise add-on.**

1. **Encryption in transit:**
   - TLS 1.3 at the edge (Cloud Armor + LB SSL policy `MODERN`, min TLS 1.2 allowed only for legacy clients, with a warning; HSTS).
   - Service-to-service traffic over HTTPS with Cloud Run IAM-authenticated invocations.
   - Kafka over TLS with SASL credentials per service.
   - Each service reaches only its own Cloud SQL instance, via the Cloud SQL connector with IAM DB auth as its own service account, TLS required.
2. **At rest:** AES-256 everywhere:
   - Google default encryption for Cloud SQL, GCS, BigQuery, Memorystore and Kafka storage.
   - CMEK with a platform key ring in asia-south1 for Cloud SQL, GCS and BigQuery.
3. **Envelope encryption with per-tenant keys:**
   - Each tenant gets a 256-bit data encryption key (DEK) per service, generated with Tink (AES-256-GCM) and wrapped by that service's key encryption key (KEK) in Cloud KMS.
   - Wrapped DEKs live in a separate **key store**: a `svc_keys` schema on its own small database with short backup retention, not the business database.
   - Unwrapped DEKs are cached in memory for ≤ 15 min.
   - Per-subject DEKs (one per employee or contact) are used for erasable PII.
   - **Enterprise add-on:** a dedicated per-tenant KMS key (SOFTWARE or HSM), or **Cloud EKM** (the customer holds the key in their own external key manager). With database per service, CMEK is set per instance (11 instances, one key per service in the platform key ring); T3 dedicated per-service instances use that tenant's CMEK.
4. **Field-level encryption** for PAN, bank account/IFSC, salary, phone/email of data principals and any government IDs:
   - Stored as ciphertext columns.
   - Search via **blind indexes**: HMAC-SHA256 under a per-tenant index key, truncated, on normalised values.
   - Masked by default in UI and API (e.g. `XXXXXX1234`). Unmasking needs a permission and is audited.
   - The OTel Collector redacts PII before logs leave the service.
   - **Aadhaar:** default is not to store the Aadhaar number at all; keep only a masked last 4 digits or a verification result. If a module truly needs it (e.g. eKYC), it goes only into a UIDAI-compliant **Aadhaar Data Vault** with reference keys and a FIPS 140-2 L3 HSM, via an ADV-as-a-service provider. Business tables hold only the reference key. Hashes are not allowed as reference keys. This is not built for MVP0/MVP1.
5. **Secrets:**
   - Secret Manager holds platform secrets only (service credentials, provider master keys), with rotation.
   - Per-tenant integration credentials (GSP, Razorpay, WhatsApp tokens) are envelope-encrypted in the owning service's DB with the tenant DEK. This keeps Secret Manager cost flat at 10k tenants.
   - The secret hard-coded in `server/gmail-bridge.gs` is rotated in W1.
6. **Tenant-isolation CI gate** (extends §8), blocking on every service PR:
   - RLS + cross-tenant API probes.
   - Cross-schema grant checks.
   - Cache-key prefix tests.
   - Kafka consumer side-effect tests.
   - Blob-path tests (`gs://…/t/{tenant_id}/`).
   - A negative test that decrypting tenant A's field with tenant B's DEK fails.
   - Nightly: the full suite against staging, plus a scheduled DAST scan.
7. **Consent manager:** a capability of identity-tenancy, exposed as `/v1/consents`.
   - Records: data principal, tenant (as data fiduciary), purpose ID, notice version, language, channel, timestamp, evidence hash and status.
   - Purposes catalogue per tenant.
   - Withdrawal is as easy as giving consent, publishes `consent.withdrawn.v1`, and consumers stop processing that purpose.
   - Notices in English and the 22 Schedule-8 languages are progressive.
   - **Interop:** the API accepts consent artefacts from a registered DPDP Consent Manager (Rule 4, from ~Nov 2026); the artefact format will follow MeitY/Board specs when published (**Unverified**). This pattern already matches Account Aggregator consent artefacts.
   - **Role mapping:** Business OS is usually the tenant's **Data Processor**; the tenant is the Fiduciary. We are Fiduciary only for our own users and leads. A DPA is signed with each tenant.
8. **India data residency:**
   - All primary data, backups, logs, Kafka and BigQuery stay in asia-south1, with DR in asia-south2. An org-policy `gcp.resourceLocations` limits resources to India regions.
   - Confluent cluster region and Grafana Cloud region must be India, or we self-host LGTM; PII is redacted before export either way (**Unverified** that India regions are offered).
   - LLM calls go through LiteLLM with PII masking, preferring India-hosted endpoints (Vertex AI asia-south1 where available).
   - Cross-border transfers are allowed under DPDP s.16 unless a country is restricted. For GDPR data subjects use SCCs and list subprocessors.
9. **Retention vs erasure:**
   - **Statutory retention:**
     - GST books and records: 72 months from the annual-return due date (CGST Act s.36).
     - Books of account: 8 years (Companies Act 2013 s.128(5)).
     - ICT logs: 180 days in India (CERT-In).
     - Statute text not re-fetched this session; confirm with a CA.
   - **DPDP:** erase personal data when its purpose ends or consent is withdrawn, unless the law requires retention.
   - **Schedule:** a retention policy per data class in a `retention_policies` table. Statutory records get a **legal hold**: restricted processing, kept until expiry, then shredded automatically.
   - **Right to erasure:** the request is verified, then `subject.erasure.requested.v1` is published. Each service deletes or anonymises the subject's rows and **destroys the subject DEK**. Tenant off-boarding: export, then destroy all of the tenant's DEKs (+ its KMS key on Enterprise).
   - **Backups:** ciphertext stays in Cloud SQL PITR/backups (7–35 days) but is unreadable once the DEK is gone. The key store has ≤ 35-day backup retention, so erasure is complete within ≤ 35 days. Restores replay an erasure ledger.
   - **Kafka:** payloads carry IDs + field-level ciphertext only, never raw PII; retention is 7 days; no log-compacted topics with PII.
   - **BigQuery:** pseudonymous IDs only, partition expiry, and DML delete on erasure.
   - **GDPR:** Art. 17 is satisfied by the same flow.
10. **Breach notification runbook** (Incident Commander = founder until a security lead is hired):

| Time from awareness | Action |
|---|---|
| T+0 | Detect (Grafana alert / report), open the incident, preserve logs, contain (rotate keys, kill switch, block tenant) |
| ≤ 6 h | **CERT-In report** (incident@cert-in.org.in, Annexure I format); CERT-In point of contact designated in advance |
| Without delay | **DPDP Board** intimation (nature, extent, timing, location, impact) + **intimation to affected data principals** in plain language via their account or registered channel (Rule 7(1)) |
| ≤ 72 h | **DPDP Board detailed report** (facts, causes, mitigation, findings on who caused it, remediation, report on intimations) (Rule 7(2)(b)); **GDPR supervisory authority** for EU subjects (Art. 33) |
| Ongoing | Notify affected tenants as processor "without undue delay" per DPA (target ≤ 24 h); post-mortem; evidence into the SOC 2 / ISO incident log |

Tabletop the runbook every quarter.
11. **Access logging and immutable audit:**
   - The **audit** service consumes all domain events plus access events: logins, PII unmask, exports, admin actions, KMS decrypt via Cloud Audit Logs Data Access.
   - Records are hash-chained (each record stores the previous hash) and exported daily to a GCS bucket with **Bucket Lock retention policy** (WORM), 180 days minimum in India.
   - Tenant admins can view their own access log at `/v1/audit`.
12. **Least privilege:**
   - One service account per service, with only its own DB user, topics, secrets and KMS KEK (`cryptoKeyEncrypterDecrypter` on that key only).
   - No human holds standing prod access. Engineers get **JIT** elevation via Google Cloud Privileged Access Manager (time-boxed, approved, logged), plus a sealed **break-glass** account (hardware-key MFA, alerting on use, post-use review).
   - Workforce SSO with MFA.
   - VPC Service Controls perimeter around data services (from MVP1).
   - IAM reviewed quarterly.
13. **Certification path (relative weeks, runs alongside story work):**
   - W1–W4: policies (ISMS scope, access, crypto, IR, retention, vendor), risk register, asset inventory; the controls above go into the templates.
   - W4–W8: pick a GRC tool (Sprinto first for INR billing and India support) and connect GCP and GitHub.
   - ~W12: first external pentest (multi-tenant scope).
   - W14–W20: **SOC 2 Type I** (Security + Confidentiality TSC) and ISO 27001 Stage 1.
   - W20–W32: SOC 2 Type II observation window (3–6 months) and ISO 27001 Stage 2 → certificate.
   - Then yearly surveillance, Type II renewal and pentest.
   - A DPDP compliance review before the May 2027 obligations apply; Significant Data Fiduciary duties (DPIA, DPO, annual audit) are not expected unless we are notified as one.

**₹ costs** (₹95.13/$; cloud unit prices from https://cloud.google.com/kms/pricing and https://cloud.google.com/secret-manager/pricing):

| Item | Unit price (sourced) | At 10k tenants / per year | Notes |
|---|---|---|---|
| KMS: per-tenant SOFTWARE key for all (option c) | $0.000082192/key-version/h ≈ $0.06/month | 10,000 × $0.06 = $600/month ≈ **₹57,100/month** | Rejected as default |
| KMS: chosen design (≈ 30 service/tier KEKs) | same | ≈ $1.80/month ≈ ₹170 | |
| KMS crypto ops | $0.03 per 10,000 | Estimate 5M unwraps/month (DEK cache) ≈ $15 ≈ ₹1,430/month | Ops volume is an **estimate** |
| KMS: Enterprise per-tenant HSM key | $0.001369863/h ≈ $1.00/month | ≈ ₹95/tenant/month | Add-on, priced into the plan |
| KMS: Enterprise EKM key | $0.004109589/h ≈ $3.00/month | ≈ ₹285/tenant/month + the customer's EKM partner fees | Add-on |
| Secret Manager (platform secrets only, ~100 versions, ~200k accesses) | $0.06/version/month (6 free); $0.03 per 10k accesses (10k free) | ≈ $5.6 + $0.57 ≈ **₹590/month** | If per-tenant creds were stored here: 30k versions ≈ $1,800 ≈ ₹1.7 lakh/month, so creds stay in the DB |
| Compliance tooling | Sprinto from $9,500/yr (platform $7,500 + first framework from $2,000) ≈ **₹9.0 lakh/yr**; Vanta Essentials from $14,000/yr ≈ ₹13.3 lakh; Drata $32,500/yr ≈ ₹30.9 lakh (AWS Marketplace list prices) | — | Indian reports put Sprinto/Scrut startup tiers at ₹2–5 lakh/yr (third-party, **estimate**). Open-source Comp AI self-host is ₹0 |
| SOC 2 audit fees | Type I ₹2–5 lakh; Type II ₹4–10 lakh; consultant-led all-in ₹2–3 lakh (Type I) / ₹3–5 lakh (Type II) | — | Third-party ranges, **estimate**; get a CPA quote |
| ISO 27001 | Certification audit ₹1.5–5 lakh; all-in programme + CB audit ₹1.5–8 lakh | — | Third-party ranges, **estimate** |
| Pentest (VAPT) | Web app ₹40k–1.5 lakh; multi-tenant SaaS ₹4–12 lakh; CERT-In empanelled +15–40% | — | Third-party ranges, **estimate**; yearly + after major changes |

Sources for the non-cloud rows:
- https://soc2auditors.org/insights/soc-2-software-pricing-comparison/
- https://soc2compliancecost.com/platforms
- https://matrixgard.com/blog/soc2-india-cost-2026/
- https://www.tcsa.in/resources/top-soc-2-consulting-firms-india
- https://www.incorpx.io/blog/startup-security-compliance-soc2-iso27001-vapt
- https://secureroot.co/en/pricing/
- https://www.tcsa.in/resources/vapt-cost-india-2026
- https://cybersigmacs.com/blog/vapt-cost-india/
- https://nofeck.in/cert-in-empanelled-vapt-provider-guide/

**Year-1 compliance budget (estimate):** about ₹10–25 lakh. That covers Sprinto at ₹2–9 lakh, the SOC 2 Type I + II audits at ₹6–15 lakh, ISO 27001 at ₹1.5–5 lakh and one multi-tenant pentest at ₹4–12 lakh. Cloud crypto runs at about ₹2–3k/month even at 10k tenants.

**Consequences:**
- The data layer gains a crypto module (Tink) and a key store from W3.
- Search on encrypted fields is limited to exact match via blind indexes; fuzzy search runs on masked or derived tokens.
- Event schemas mark PII fields (`x-pii: true`), and CI rejects raw PII in event payloads.
- Erasure and retention are testable features, with an e2e test that erases a subject and checks every service.
- Sources (other): https://gdpr-info.eu/art-33-gdpr/ ; https://gdpr-info.eu/art-17-gdpr/ ; https://cloud.google.com/kms/docs/ekm ; https://developers.google.com/tink ; https://cloud.google.com/storage/docs/bucket-lock ; https://cloud.google.com/iam/docs/pam-overview

Next ADRs after these: identity and SSO (Identity Platform → Keycloak), API gateway product choice (KrakenD vs Envoy vs Google API Gateway), AI agent permissions and evals, mobile/offline (PWA queue).



## Related Jira epics

- [SCRUM-9](https://rozvaa.atlassian.net/browse/SCRUM-9)
- [SCRUM-84](https://rozvaa.atlassian.net/browse/SCRUM-84)
- [SCRUM-5](https://rozvaa.atlassian.net/browse/SCRUM-5)


_Source: [Architecture v1 Google Doc](https://docs.google.com/document/d/1sKoivvz3JLkKPI1Jta0i6XWnWuW57-rSwWICmm1w9VM/edit)_
