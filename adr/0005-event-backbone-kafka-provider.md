# ADR-005 Event backbone: Kafka provider
- **Context:** async-first design. Requirements:
  - per-tenant ordering
  - replay
  - schema registry
  - sinks to BigQuery
  - India-region hosting
  - scale-to-zero cost at MVP0
  - a solo or small team, so maintainability matters most
- **Options, with sourced prices (₹ at 95.13/$):**

| Option | MVP0 | MVP1 | ~10k tenants | Maintainability | Source |
|---|---|---|---|---|---|
| **Confluent Cloud** | Basic: first eCKU free, then $0.14/eCKU-h; $0.05/GB in/out; $0.08/GB-month stored → ≈ ₹0–500 | Basic ≈ ₹500–1,000 | Standard from ~$385/month (≈ ₹36,600), $0.75/eCKU-h; at a few eCKUs plus traffic, ≈ ₹1.5–2.6 lakh/month (**estimate**) | Fully managed, serverless, scales to zero; Schema Registry and BigQuery sink connector available; highest unit price at scale | https://www.confluent.io/pricing/ ; https://docs.confluent.io/cloud/current/billing/billing-dimensions.html |
| **Aiven for Kafka** | Free tier (5 topics, no SLA) or Developer $35/month ≈ ₹3,330 (replication factor 1, 99% SLA, ≤20 topics) | Professional from $180/month ≈ ₹17,100 | Plan-based; calculator example $274/month at low throughput; higher plans quote-based (**Unverified**) | Managed, runs on GCP Mumbai; predictable plans; topic limits on small plans clash with topic-per-event | https://aiven.io/kafka ; https://aiven.io/docs/products/kafka/dev-tier/kafka-dev-tier |
| **Redpanda Cloud (Serverless)** | Pay-as-you-go, $0 base; unit rates only in the calculator (**Unverified**) | **Unverified** | A vendor-quoted example was ~$335/month (≈ ₹31,900) per cluster for 0.2 MB/s with PrivateLink and 30-day retention (Redpanda employee on Reddit; indicative only) | Kafka-API compatible, built-in Schema Registry; fewer managed connectors | https://www.redpanda.com/pricing ; https://www.reddit.com/r/apachekafka/comments/1sueyj4/ |
| **Strimzi on GKE (self-hosted)** | GKE cluster fee + ≥3 broker nodes; ≈ ₹15–25k/month (**estimate**; GKE fee and node prices not re-checked this session) | same, plus ops time | Cheapest per GB at scale, but needs a Kafka/K8s operator on call | Highest ops burden: upgrades, disks, rebalancing, Schema Registry and Connect all self-run; contradicts lean solo operations | https://strimzi.io ; https://cloud.google.com/kubernetes-engine/pricing |

- **Decision:** **Confluent Cloud.**
  - Basic cluster for dev, staging and prod through MVP1 (≈ ₹0–1,000/month).
  - Move prod to Standard (99.99% SLA, ~$385/month starting) when paying volume or SLA needs it (≈ MVP2, or about 100 tenants).
  - Use only the plain Kafka API + Schema Registry so Aiven/Redpanda stay an exit path.
  - Re-evaluate at ~1,000 tenants, when Standard costs exceed Aiven Professional or BYOC by more than 2×.
  - **Check before use:** that Basic/Standard clusters are offered in GCP asia-south1 (**Unverified**).
- **Consequences:**
  - Pub/Sub is not used.
  - Scale-to-zero Cloud Run services need the event-dispatcher (worker pool) to consume for them.
  - The outbox relay is a poller first, Debezium later.
  - Per-tenant partition keys; topic ACLs per service.



## Related Jira epics

- [SCRUM-8](https://rozvaa.atlassian.net/browse/SCRUM-8)


_Source: [Architecture v1 Google Doc](https://docs.google.com/document/d/1sKoivvz3JLkKPI1Jta0i6XWnWuW57-rSwWICmm1w9VM/edit)_
