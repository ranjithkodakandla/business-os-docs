# Integrations

## 10. Integration architecture

Each integration is an adapter **inside the owning service**:

| Integration | Owning service |
|---|---|
| GSP (e-invoice, e-way bill, GSTR) | billing-invoicing |
| Razorpay, RazorpayX, Account Aggregator | collections-payments |
| WhatsApp, email, SMS | notifications |
| LLM | ai-agents, via LiteLLM |
| Tally | its own small service from MVP1 (bridge agent → gateway) |
| Biometric devices | people/attendance service (MVP2) |

- Each adapter has `real` and `simulated` implementations.
- Outbound calls are driven by outbox commands with idempotency keys.
- Inbound webhooks land at the gateway, are signature-verified, written to the owning service's inbox table, and processed from there.
- Per-tenant credentials live in Secret Manager, encrypted with KMS.
- Retries with backoff, plus a DLQ.

| Integration | Providers (primary first) | Current ₹ pricing | Status | Source | MVP |
|---|---|---|---|---|---|
| GST e-invoice (IRN) | Masters India, ClearTax, IRIS, Adaequare (GSPs); NIC IRP sandbox | GSPs ~₹0.25–1.50 per IRN; Masters India quote-only; ClearTax from ₹7,777/mo | Range verified in the plan Sheet; Masters India **Unverified** (quote) | Plan Sheet Cost projections; https://www.mastersindia.co ; https://einvoice1.gst.gov.in | MVP1 |
| E-way bill | Same GSP | Usually bundled; Sheet assumes ₹0.50/EWB | **Unverified** (estimate) | Plan Sheet | MVP1 |
| GSTR-1/3B API | ClearTax, Masters India | No public price; Sheet assumes ₹50/GSTIN/month | **Unverified** | Plan Sheet | MVP1 |
| GSTIN verification | API Sathi, GSTINAPI, GSP | ₹0.55/call (API Sathi); ₹0.40–0.80 (GSTINAPI) | Verified (Sheet) | Plan Sheet | MVP0 |
| PG / UPI collections | Razorpay, Cashfree, PayU | Razorpay 2% platform fee on all domestic instruments; UPI has zero MDR (the 2% is a tech fee); +18% GST; no setup/AMC | Verified | https://razorpay.com/pricing/ | MVP1 (tenant collections are pass-through) |
| Subscriptions / e-mandate | Razorpay Subscriptions | 2% + GST; subscription cards +0.9% | Verified (Sheet) | https://razorpay.com/pricing/ | MVP1 |
| Payouts | RazorpayX | BB+ Core ₹2,476/quarter; IMPS/UPI ₹4 (<₹1k) / ₹6 (₹1k–25k) / ₹9 (>₹25k); NEFT/RTGS ₹5; +GST | Verified | https://razorpay.com/x/payouts_x/ | MVP1–2 |
| Bank feeds | Account Aggregator via TSP (Finvu, Setu, OneMoney); fallback statement upload | Finvu ₹0.20–2.50/fetch; new per-fetch model from 1 Dec 2026 (per Sheet) | Verified (Sheet); TSP fees **Unverified** | Plan Sheet; https://sahamati.org.in | Statement upload MVP1; AA MVP3 (start W10) |
| WhatsApp | Meta Cloud API direct, or a BSP: Gupshup, Interakt, AiSensy; Twilio (prototype) | Meta India ₹0.8631 marketing / ₹0.115 utility-auth-service (1,000 free service msgs/number/month). Gupshup ₹1.15 / ₹0.145, no subscription. Interakt Starter ₹3,499/quarter + ₹0.970 marketing. AiSensy ₹1,500/mo + ₹1.09 / ₹0.145. Twilio $0.005/msg + Meta | Verified (vendor pages, Oct 2026) | https://developers.facebook.com/documentation/business-messaging/whatsapp/pricing ; https://www.gupshup.ai/rate-card ; https://aisensy.com/pricing ; https://www.interakt.shop ; https://www.twilio.com/en-us/whatsapp/pricing | MVP0 sandbox; prod after Meta verification (W6–W8) |
| SMS / voice | Twilio / a DLT-registered provider | DLT entity ₹5,900 incl GST, one-time | Verified (Sheet) | Plan Sheet | MVP0 |
| Tally | Own connector: Tally XML over HTTP via a small bridge agent on the customer PC that connects outbound to the gateway | ₹0 vendor fee; customer's own Tally Silver ₹750/mo, Gold ₹2,250/mo | Verified (Sheet) | https://help.tallysolutions.com/integrate-with-tallyprime/ | MVP0 import; MVP1 sync |
| Biometric | eSSL / ZKTeco ADMS push; Matrix COSEC API | ₹0 to us for customer-owned devices; hardware resale prices not checked | ₹0 verified (Sheet); hardware **Unverified** | Plan Sheet HW1/HW2 | MVP2 / MVP3 |
| LLM | LiteLLM proxy (OSS) → Gemini Flash; OpenAI, Sarvam and NVIDIA as fallbacks | Gemini Flash $0.75 / $3.75 per 1M input/output tokens (doubles from 1 Jan 2027 per the Risks tab); LiteLLM ₹0 + hosting | Verified (Sheet) | https://ai.google.dev/gemini-api/docs/pricing ; https://github.com/BerriAI/litellm | MVP0 |
| OCR of bills | Google Document AI / Gemini | Invoice parser $0.10/doc (≤10 pages) | Verified (Sheet) | Plan Sheet | MVP1 |



## Related Jira epics

- [SCRUM-72](https://rozvaa.atlassian.net/browse/SCRUM-72)
- [SCRUM-17](https://rozvaa.atlassian.net/browse/SCRUM-17)
- [SCRUM-75](https://rozvaa.atlassian.net/browse/SCRUM-75)
- [SCRUM-77](https://rozvaa.atlassian.net/browse/SCRUM-77)
- [SCRUM-76](https://rozvaa.atlassian.net/browse/SCRUM-76)
- [SCRUM-79](https://rozvaa.atlassian.net/browse/SCRUM-79)
- [SCRUM-88](https://rozvaa.atlassian.net/browse/SCRUM-88)
- [SCRUM-14](https://rozvaa.atlassian.net/browse/SCRUM-14)
- [SCRUM-18](https://rozvaa.atlassian.net/browse/SCRUM-18)
- [SCRUM-80](https://rozvaa.atlassian.net/browse/SCRUM-80)


_Source: [Architecture v1 Google Doc](https://docs.google.com/document/d/1sKoivvz3JLkKPI1Jta0i6XWnWuW57-rSwWICmm1w9VM/edit)_
