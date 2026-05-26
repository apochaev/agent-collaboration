# Meridian Platform — Architecture Notes
_Maintained by Priya Nair, Lead Architect. Last updated: 2025-12-10._

> These notes are a working reference, not a specification. For authoritative service contracts,
> see the OpenAPI specs in each service's `/docs` directory. For infrastructure state, consult
> Terraform in the `platform-infra` repo.

---

## System Overview

Meridian is a data observability platform. Customers connect their data pipelines by sending events
to our ingestion surface; we validate those events against customer-defined schemas and freshness
rules, detect anomalies, and deliver alerts through their preferred channels (Slack, PagerDuty,
email, or webhook).

The platform processes roughly 4–6 billion events per day in production. Peak throughput is around
80,000 events/second, which arrives in bursts aligned with customer ETL job windows (most commonly
2–4 AM and 6–8 AM UTC).

Data flows through six backend services. Most coordinate through Kafka; the scheduler runs on a
periodic loop. There is a frontend for customer self-service and an internal admin surface for
support and ops tooling.

---

## Services

### collector-svc (legacy) and ingestion-api (current)

Events enter the system through one of two surfaces. The original `collector-svc` was a stateful
TCP + gRPC service that held long-lived customer connections and batched events before forwarding.
It was effective for early customers on high-volume dedicated pipelines, but it was difficult to
operate: it required dedicated node pools, didn't scale horizontally well under bursty load, and
had a complex session management layer that caused problems during rolling deployments.

`ingestion-api` is the current entry point. It is a stateless Spring Boot REST service that accepts
events via a single POST endpoint, validates auth, does minimal structural validation (schema ID
exists, event is not malformed JSON), and publishes to the `events.raw` Kafka topic. It is
deployed as a standard K8s Deployment with HPA configured to scale on request rate.

Both surfaces produce events on `events.raw`. The message envelope format is identical.

Migration to `ingestion-api` has been underway for 18 months. As of this writing, roughly 80% of
event volume is on `ingestion-api`. Legacy customers on long-term contracts remain on `collector-svc`
in maintenance mode. Do not make changes to `collector-svc` without consulting the Core team — the
remaining customer migrations are sensitive and the service is in a deliberately stable state.

### processor-svc

Kafka consumer on `events.raw`. Responsible for the core domain logic:

- **Schema validation**: looks up the customer's registered schema version and validates each event
  field by field.
- **Enrichment**: attaches customer metadata (tier, timezone, shard key) that is needed downstream.
- **Freshness state management**: updates the per-dataset freshness timestamp and compares against
  the customer's configured SLA.
- **Anomaly detection (volume)**: compares current event rate against a rolling 14-day window using
  an EWM algorithm. Triggers volume alerts when the deviation exceeds the configured sensitivity
  threshold.
- Writes validated events to the `events.validated` topic.
- Writes alert signals to the `alerts.triggered` topic when anomalies or freshness violations are
  detected.

`processor-svc` owns the `events`, `datasets`, and `alert_signals` tables in PostgreSQL. No other
service writes to these tables.

This is the most critical service in the system. Latency in `processor-svc` degrades the freshness
signal accuracy and delays alert delivery. The P99 processing latency SLA is 800ms end-to-end from
event receipt to alert trigger.

### query-api

REST API that powers both the customer-facing dashboard and the external API (documented at
`docs.meridian.io`). All reads. No writes.

Queries against the PostgreSQL read replica. Most queries are time-bounded (last N hours or days)
so query plans are generally stable. The one exception is dataset lineage queries, which can become
expensive when customers have deep pipeline graphs; these are gated behind a slower dedicated
endpoint with a separate connection pool.

Key endpoints:
- `GET /v1/datasets/{id}/status` — current freshness and schema status
- `GET /v1/datasets/{id}/events` — paginated event history
- `GET /v1/alerts` — alert feed with filtering
- `GET /v1/reports/summary` — aggregate dashboard data

The external API uses API key auth. The internal dashboard uses session JWT.

### scheduler-svc

Periodic job runner responsible for:

- Evaluating time-based freshness rules on a schedule (e.g., "this dataset should receive at
  least one event every 6 hours").
- Running end-of-window volume checks for customers using window-based anomaly detection.
- Orchestrating daily report generation.
- Pruning stale alert records older than the customer's configured retention window.

Jobs are defined as Quartz `Job` implementations and registered in a Quartz `Scheduler`. The job
store is database-backed using PostgreSQL, so job state survives restarts. Multiple scheduler-svc
instances run in production; Quartz clustering handles job deduplication via database row locking.
The Quartz version in use is 2.3.2.

scheduler-svc has no consumer role on Kafka but it does produce: when a time-based rule fires,
it publishes an alert signal to the `alerts.triggered` Kafka topic, which notification-svc
consumes alongside processor-svc-generated signals.

### notification-svc

Responsible for alert delivery. Consumes the `alerts.triggered` Kafka topic.

For each alert signal:
1. Looks up the customer's configured notification channels.
2. Deduplicates: if an identical alert was delivered within the deduplication window (configurable
   per customer, default 15 minutes), it is suppressed.
3. Delivers to each configured channel: Slack webhook, PagerDuty events API, email (via SendGrid),
   or outbound webhook.
4. Records delivery status and any retry state.

notification-svc is designed to be eventually consistent with respect to delivery. It will retry on
transient failures with exponential backoff up to 3 hours. After that, the alert is marked as
failed and a support event is created.

### admin-api

Internal-only service for the support and operations team. Not customer-facing.

Provides tooling for: customer account management, manual alert suppression, schema registry
debugging, pipeline diagnostics. Authentication is entirely separate from the customer-facing auth:
admin users authenticate via the company SSO (Okta), and the admin-api validates JWTs issued by
Okta rather than the customer auth service.

The admin-api is not covered by the public API SLA. Support team should expect occasional restarts
and planned maintenance windows.

### frontend

React 18 / TypeScript single-page app. Served from GCS CDN. Talks only to `query-api` (reads) and
`ingestion-api` for the onboarding wizard (event stream setup). No direct Kafka or database access.

The frontend repo is separate from the core monorepo: `meridian/frontend` on GitHub.

---

## Data Store

PostgreSQL 13, managed on CloudSQL. We run one primary and one read replica per region
(us-central1 for production, us-east1 for DR). Connection pooling via PgBouncer (session mode for
OLTP, statement mode for analytics queries).

Database schema is partitioned per tenant using PostgreSQL schemas (not separate databases). Each
customer gets a schema named `tenant_<customer_id>`. The public schema holds shared reference data:
the schema registry, customer account records, and system configuration.

Tables of significance:
- `public.customers` — account records, subscription tier, feature flags
- `public.schema_registry` — registered schemas and their versions
- `tenant_<id>.events` — raw validated events (partitioned by day, retained 90 days)
- `tenant_<id>.datasets` — dataset metadata and current status
- `tenant_<id>.alert_signals` — alert history and delivery state

PostgreSQL access is granted via role. Request access in `#data-access`; typical SLA is 1–2
business days. Credentials are issued from HashiCorp Vault.

---

## Message Broker

Apache Kafka, managed on Confluent Cloud. Three topics:

| Topic             | Producers                  | Consumers        | Retention |
|---|---|---|---|
| `events.raw`      | ingestion-api, collector-svc | processor-svc  | 24 hours  |
| `events.validated`| processor-svc              | (analytics, future) | 7 days |
| `alerts.triggered`| processor-svc, scheduler-svc | notification-svc | 72 hours  |

Partition counts: `events.raw` has 120 partitions (sized for peak throughput). Other topics have 24.

Schema registry: Confluent Schema Registry. All Kafka messages use Avro. Adding a new schema or
evolving an existing one requires a schema compatibility check — this is enforced by the registry.

Do not create new Kafka topics without a Design Review. Topic creation is gated on a Terraform PR
to the `platform-infra` repo.

---

## Infrastructure

Kubernetes on GKE. Three clusters: `prod` (us-central1), `staging` (us-central1), `dev`
(us-central1). Each cluster has multiple node pools: a standard pool for most services and a
high-memory pool for processor-svc.

CI/CD: GitHub Actions for build and test; ArgoCD for deployment. The deployment manifests live in
the `platform-infra` repo under `k8s/`. ArgoCD watches the manifests and applies them automatically
to staging after a successful merge to `main`. Production promotion is manual (ArgoCD sync with a
required approval gate).

Service mesh: Istio is planned for Q1 2026. Currently, inter-service communication uses internal
GKE service discovery (cluster-internal DNS). mTLS is not yet enforced between services.

Secrets management: HashiCorp Vault (self-hosted). Services pull secrets at startup via the Vault
agent sidecar. Do not hardcode secrets. Do not use Kubernetes Secrets directly for sensitive values.

Observability: Datadog for APM, distributed tracing, and centralized logs. Internal Grafana
dashboard at `grafana.meridian.internal` for infrastructure metrics (requires VPN).

---

## Known Issues and Tech Debt

- **collector-svc migration**: The migration from collector-svc to ingestion-api is ongoing. Do not
  make changes to collector-svc without consulting the Core team.
- **Quartz clustering edge case**: The Quartz job store in scheduler-svc has occasional job
  duplication under high contention during deployment rollouts. Mitigation: avoid deploying
  scheduler-svc during job-heavy windows (typically 2–4 AM UTC).
- **Lineage query performance**: Dataset lineage queries in query-api degrade quadratically on deep
  graphs. Ticket MER-3421 tracks moving this to a graph database. Not on H1 roadmap.
- **Schema migration safety**: We use Flyway for database migrations. Destructive migrations
  require a review from the Core team and a documented rollback plan.
