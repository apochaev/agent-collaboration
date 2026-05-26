# Meridian Systems — First-Week Engineering Onboarding

_Synthesized by Claude from three internal documents: `arch-notes.md` (Priya Nair, Dec 2025),
`team-conventions.md` (Andy K., Mar 2026), and `slack-qa.md` (Feb–Apr 2026 export). Where sources
contradict each other, this guide resolves the contradiction and names the evidence. Where facts
can't be verified, a flag explains why._

---

## What Is Meridian?

Meridian is a **data observability platform**. Customers connect their data pipelines to us by
sending events. We check those events against rules the customer has defined (schema correctness,
freshness — did the data arrive on time?) and alert them when something looks wrong.

The system processes roughly **4–6 billion events per day**, with peaks around 80,000 events per
second. Most traffic arrives in bursts aligned with customer ETL jobs (commonly 2–4 AM and 6–8 AM
UTC). If you're on call or debugging latency, those windows are when things get interesting.

---

## First-Day Checklist

Complete these on or before your first day. Some have dependencies that take days to clear.

- [ ] **Datadog access** — Ask your manager to add you to the Datadog org. This is manual; it
  doesn't auto-provision. You need this for logs and APM from day one.

- [ ] **Confluence SSO** — File an IT ticket at `it-help.meridian.internal` (this site works
  without SSO). It usually takes 2–3 business days. You need Confluence for RFC templates and
  runbooks. If you need the RFC template before SSO clears, ask Andy K. — he can DM it to you.

- [ ] **VPN access** — Internal tools (Grafana, ArgoCD, Vault) are behind VPN. Check with IT or
  your manager for how to get credentials.
  > **Unverified:** The source documents mention VPN is required but don't describe how to set it
  > up. Ask your manager or in `#new-hires` on your first day.

- [ ] **Join Slack channels** — At minimum: `#eng-general`, `#new-hires`, `#code-review`,
  `#incidents`, `#platform-rfc`, `#data-access`, `#core-domain-reviews`.

- [ ] **Request DB access if needed** — If your first task involves looking at production data,
  post in `#data-access` with your name, team, what you need access to, and the Jira ticket. Tag
  **Leila Mohr** as approver. Expect 1–2 business days. Credentials come from Vault once approved.

- [ ] **Clone the repos** — There are two:
  - `meridian/core-backend` — the main monorepo (all backend services)
  - `meridian/frontend` — the React app (separate repo)

- [ ] **Run the dev stack** — See [Dev Setup](#dev-setup) below.

---

## How the System Works

Events flow through six backend services, connected primarily via Kafka:

```
Customer
   │
   ▼
ingestion-api          ← stateless REST, auth check, minimal validation
   │
   │ publishes to Kafka: events.raw
   ▼
processor-svc          ← schema validation, enrichment, anomaly detection
   │
   ├─ publishes to Kafka: events.validated (analytics, future use)
   └─ publishes to Kafka: alerts.triggered (when anomaly or freshness violation detected)
                                │
                    ┌───────────┴────────────┐
                    ▼                        ▼
             scheduler-svc          notification-svc
           (time-based rules)    (delivers to Slack, PD, email, webhook)

query-api  ← read-only; powers the customer dashboard and public API
admin-api  ← internal only; support tooling (Okta SSO)
```

**The most important service is `processor-svc`.** It owns the core domain logic and the most
critical tables. Its P99 latency SLA is 800ms end-to-end from event receipt to alert trigger. If
you're debugging alert delays, start here.

---

## Services

### ingestion-api

The current and **only** entry point for customer events. A stateless Spring Boot REST service.
Accepts events via a single POST endpoint, validates auth, checks that the event is valid JSON and
references a known schema ID, then publishes to the `events.raw` Kafka topic. Scales horizontally
via Kubernetes HPA on request rate.

> **Contradiction resolved — collector-svc is gone:** The December 2025 architecture notes
> describe a legacy service called `collector-svc` as still active for ~20% of traffic. In a
> March 2026 Slack thread, Leila Mohr (platform team) clarified that collector-svc was **fully
> decommissioned in January 2026** — the last migration batch completed during the holiday week.
> Tom Weston initially thought one customer remained on the old path but confirmed Leila was
> correct. `ingestion-api` is the only entry point. If you see references to `collector-svc`
> anywhere, treat them as outdated.

### processor-svc

Consumes from `events.raw`. Runs:
- Schema validation (field-by-field against the customer's registered schema version)
- Enrichment (attaches customer metadata needed downstream)
- Freshness state management (compares event timestamps against customer-configured SLAs)
- Volume anomaly detection (14-day rolling window, EWM algorithm)

Writes alerts to `alerts.triggered` when it detects a problem. Owns the `events`, `datasets`, and
`alert_signals` tables. No other service writes to these tables.

### query-api

Read-only REST API. Powers both the customer-facing dashboard and the external API documented at
`docs.meridian.io`. Queries against the PostgreSQL read replica. The external API uses API key
auth; the internal dashboard uses session JWT.

One expensive endpoint: dataset lineage queries (`GET /v1/datasets/{id}/lineage`). These degrade
on deep pipeline graphs and are isolated to a dedicated endpoint with its own connection pool.
Ticket MER-3421 tracks moving this to a graph database; it's not on the H1 roadmap.

### scheduler-svc

Runs periodic jobs: time-based freshness rule evaluation, end-of-window volume checks, daily
report generation, stale alert pruning.

> **Contradiction resolved — Quartz is gone:** The December 2025 architecture notes describe
> scheduler-svc as using Quartz 2.3.2 with a PostgreSQL-backed job store. In an April 2026 Slack
> thread, Tom Weston (who handles day-to-day maintenance of this service) confirmed that Quartz
> was migrated out in March 2026. The service now uses **Spring `@Scheduled` with Redis
> distributed locking** to prevent duplicate execution across instances. If you're adding a new
> scheduled job, look at the existing `@ScheduledJob` beans in the `scheduler-svc` package and
> read the javadoc on `RedisJobLock.java`. Lock keys follow the pattern `scheduler:<job-name>`;
> TTL is configurable per job via `@ScheduledJob` annotation parameters.

### notification-svc

Consumes from `alerts.triggered`. For each signal: looks up the customer's notification channels,
deduplicates (default 15-minute window per customer), and delivers to Slack webhook, PagerDuty,
email (via SendGrid), or outbound webhook. Retries on transient failure with exponential backoff
for up to 3 hours before marking delivery as failed and creating a support event.

### admin-api

Internal-only. Used by the support and operations team for account management, manual alert
suppression, schema registry debugging, and pipeline diagnostics. Authenticated via Okta SSO — not
the customer-facing auth system. Not covered by the public API SLA.

### frontend

React 18 / TypeScript single-page app. Lives in `meridian/frontend` (separate repo from
`core-backend`). Talks to `query-api` for reads and `ingestion-api` for the onboarding wizard.
Served from GCS CDN.

---

## Data Stores

### PostgreSQL

Version 13, managed on CloudSQL. One primary + one read replica in us-central1 (production) and
us-east1 (disaster recovery). Connection pooling via PgBouncer.

**Tenant isolation**: Each customer has their own PostgreSQL schema named `tenant_<customer_id>`.
The public schema holds shared data: schema registry, customer accounts, system config.

Key tables:
- `public.customers` — account records, subscription tier, feature flags
- `public.schema_registry` — event schemas and their versions
- `tenant_<id>.events` — validated events (partitioned by day, retained 90 days)
- `tenant_<id>.datasets` — dataset metadata and current status
- `tenant_<id>.alert_signals` — alert history and delivery state

Database migrations use Flyway. Destructive migrations require a Core team review and a documented
rollback plan.

> **Unverified — PostgreSQL version:** The architecture notes specify PostgreSQL 13 as of December
> 2025. No source confirms whether this has been upgraded since. Before planning work that depends
> on a specific Postgres version feature, verify with the platform team.

### Kafka

Managed on Confluent Cloud. Three topics:

| Topic               | Who writes         | Who reads         | Retention |
|---------------------|--------------------|-------------------|-----------|
| `events.raw`        | ingestion-api      | processor-svc     | 24 hours  |
| `events.validated`  | processor-svc      | (analytics, future) | 7 days  |
| `alerts.triggered`  | processor-svc, scheduler-svc | notification-svc | 72 hours |

All Kafka messages use Avro with Confluent Schema Registry. Evolving a schema requires a
compatibility check enforced by the registry. **Do not create new topics without a Design Review**
— topic creation requires a Terraform PR to `platform-infra`. This is separate from the RFC
process (see [RFC Process](#rfc-process)).

---

## Dev Setup

**Use `make dev` in the `core-backend` repo root.** Do not use `docker-compose up` directly — it
starts containers but services will fail on startup because the Vault configuration isn't
injected. `make dev` wraps docker-compose and additionally:

- Injects the dev Vault token so services can pull secrets
- Sets up local Kafka with the correct topic configuration
- Seeds test customer data so the UI is usable end-to-end

First run takes a while — it pulls several Docker images. Subsequent runs are much faster.

If you're working on a single service in isolation and you know what you're doing, `docker-compose
up -d` inside the service directory works, but you'll lose the test data seeding. As a new
engineer, use `make dev`.

For Vault credentials in development, `make dev` handles provisioning. The Vault UI is at
`vault.meridian.internal` (VPN required).

---

## Key People

| Name | Role | When to contact |
|------|------|-----------------|
| **Priya Nair** | Lead Architect | Architectural decisions, structural changes to `core-domain`, `#platform-rfc` |
| **Tom Weston** | Senior Engineer (Core team) | Day-to-day `core-domain` reviews, scheduler-svc questions; he escalates to Priya when needed |
| **Leila Mohr** | Platform team | DB access requests (#data-access), RFC process, CI/infra questions — authoritative voice on platform policies |
| **Andy K.** | Senior Engineer | Team conventions questions, RFC templates, local dev help, code review |
| **Dan Reyes** | Engineer | General engineering — note: was behind on the CI coverage gate change; for platform policy questions, prefer Leila |

> **Note on core-domain reviews:** The team conventions doc (written March 2026) says to ask Priya
> for all core-domain changes. A concurrent Slack thread clarifies: Tom took over regular
> core-domain reviews when Priya moved to a full architecture role. **For new domain events or
> changes within existing aggregates, tag Tom as reviewer and mention in `#core-domain-reviews`.**
> Priya stays involved for structural changes to the domain model. When in doubt, ping Tom first
> — he'll tell you if Priya needs to be in the loop.

---

## Making Changes

### Branching

Use `initials/ticket-slug` format:

```
ak/MER-1234-fix-auth-token-refresh
tw/MER-1456-add-dataset-lineage-endpoint
```

A Jira ticket is required — make one if none exists. Both `initials/` and `username/` formats
appear in the repo history; the initials format is current convention. Stale branches are pruned
monthly.

### Pull Requests

- Two approvals required. Exception: doc changes, config tweaks, and dependency bumps that don't
  affect runtime behavior can merge with one. Use judgment; when in doubt, get two.
- Open as a draft early if you want approach feedback. Don't request reviews until it's ready.
- Squash merge is the default.
- Ping `#code-review` (not DMs) if you're blocked waiting for a review.
- Expect a first pass within 24 hours; if you're on a deadline, say so in the PR description.

### RFC Process

File an RFC before writing code if your change:
- Adds or modifies an external API endpoint (including adding an **optional** field — some
  customers parse responses strictly and this matters)
- Changes a Kafka message schema in a way that affects cross-service contracts
- Affects how customers integrate with Meridian

RFC template is on Confluence (Engineering → RFCs → Template). If your Confluence SSO isn't
active yet, ask Andy K. — he can DM you the template.

After filing, @mention the relevant team leads. Review SLA is 5 business days.

If you're not sure whether something needs an RFC, ask in `#platform-rfc`. You'll get an answer
in about 15 minutes.

**Kafka topics are separate:** A new Kafka topic requires a Design Review and a Terraform PR to
`platform-infra`, not an RFC. If a topic is part of a cross-service API change, you file the RFC
for the contract and the Terraform PR for the topic — these run in parallel.

### Deployments

**Staging** is automatic. Merging to `main` triggers an ArgoCD sync to staging. No approval needed.
If staging breaks after your merge, you're responsible for fixing it or rolling back before anyone
else merges.

**Production** is manual. Open ArgoCD at `argocd.meridian.internal` (VPN required), find your
service, click Sync. This triggers an approval gate requiring a thumbs-up from your team lead or
the on-call engineer. Deploy during business hours (9 AM–5 PM PT).

**Do not deploy on Fridays.** This is a cultural norm, not a hard block, but you will hear about it
if you merge something risky on a Friday afternoon.

**Rollbacks** are fast in ArgoCD — point to the previous image tag and sync. For user-facing
impact: roll back first, investigate after.

---

## Testing

### Coverage Gate

85% line coverage per module is enforced by CI via JaCoCo. This is a hard gate — your PR will
fail if coverage drops below 85%. It is not optional.

> **Contradiction resolved — coverage is enforced:** The team conventions doc describes 85% as a
> "target" (implying soft). In an April 2026 Slack thread, Dan Reyes said it wasn't enforced and
> suggested just adding a note. Leila Mohr immediately corrected him: the platform team enabled
> JaCoCo threshold enforcement in CI in Q1 2026, announced in `#eng-general` on March 10. Dan
> acknowledged he had missed the announcement. **The 85% threshold is a hard CI gate.**

### Test Levels

Three levels, all required:

**Unit tests** — Test individual classes in isolation. Mock everything external. Run with
`./gradlew test`. These are fast and run on every build.

**Integration tests** — Tagged `@IntegrationTest`. Hit a real database or message broker.
Most use H2 in-memory; ones that need real PostgreSQL behavior use Testcontainers. Run separately
with `./gradlew integrationTest` (excluded from the default test task because they're slow). CI
runs them on every PR. See `processor-svc/src/test/java/io/meridian/processor/integration/` for
examples and `BaseIntegrationTest.java` for the setup pattern.

**Contract tests** — Spring Cloud Contract for Kafka schema contracts. Required for any change to a
Kafka message schema. If you touch Avro schemas in `shared/`, you need contract tests.

**E2E tests** — Not yet implemented. For now: integration tests + manual verification in staging
before promoting to prod.

---

## Observability

| Tool | Access | What it's for |
|------|--------|---------------|
| Datadog | Ask manager day one | APM, distributed tracing, centralized logs |
| Grafana | `grafana.meridian.internal` (VPN) | Infrastructure metrics |
| ArgoCD | `argocd.meridian.internal` (VPN) | Deployment management |

Key Datadog dashboards:
- "Ingestion API - Production" — ingestion-api metrics
- `grafana.meridian.internal/d/ingestion-overview` — infra view for ingestion (VPN required)

---

## Known Issues and Tech Debt

**Lineage query performance:** Dataset lineage queries in `query-api` degrade quadratically on
deep pipeline graphs. They're isolated to a dedicated endpoint with a separate connection pool.
Ticket MER-3421 tracks moving this to a graph database; not on the H1 roadmap. Avoid running
lineage queries in tight loops.

**Quartz scheduler removed:** The architecture notes reference Quartz 2.3.2 in scheduler-svc.
This is outdated. The migration to Spring `@Scheduled` + Redis locking completed in March 2026.
If you see Quartz references in old code, they're dead weight.

**Istio / mTLS:** The December 2025 architecture notes say Istio was planned for Q1 2026. As of
the last available source (April 2026), no document confirms it shipped. Inter-service
communication currently uses cluster-internal GKE DNS without mTLS.

> **Unverified — Istio status:** No source after December 2025 confirms whether Istio was
> deployed on schedule. If your work involves service-to-service auth or traffic policies, verify
> current state with the platform team before assuming mTLS is available or unavailable.

---

## Appendix: Contradictions Resolved

This section documents how conflicts were resolved, with sources named.

| Claim in source docs | Resolved to | Evidence |
|---|---|---|
| collector-svc still active for ~20% of traffic (`arch-notes.md`, Dec 2025) | collector-svc was fully decommissioned in January 2026 | Leila Mohr, Slack Thread 4, March 2026. Tom Weston initially disagreed but confirmed her correction. |
| 85% coverage is a "target" (`team-conventions.md`) | 85% is a hard CI gate | Leila Mohr, Slack Thread 6, April 2026. Announced in `#eng-general` March 10, 2026. Dan Reyes was behind on this. |
| scheduler-svc uses Quartz 2.3.2 (`arch-notes.md`, Dec 2025) | Quartz was removed in March 2026; now Spring `@Scheduled` + Redis locking | Tom Weston, Slack Thread 8, April 2026. |
| core-domain reviews go to Priya (`team-conventions.md`) | Day-to-day reviews go to Tom Weston; Priya handles structural/architectural changes | Andy K. and Tom Weston, Slack Thread 5, April 2026. |
