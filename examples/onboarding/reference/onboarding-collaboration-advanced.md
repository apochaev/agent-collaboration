# Meridian Systems — First-Week Engineer Onboarding Guide

_Synthesized from: arch-notes.md (Priya Nair, Dec 2025), team-conventions.md (Andy K., Mar 2026), and #eng-questions Slack export (Feb–Apr 2026). Where sources conflict, this guide names the resolution and its evidence. Where claims cannot be verified from those sources, this guide explains why._

---

## What Meridian Does

Meridian is a **data observability platform**. Customers connect their data pipelines by sending events (structured records about data arriving, transforming, or moving) to Meridian's API. Meridian checks those events against rules the customer sets — things like "this dataset should receive at least one new record every 6 hours" — and sends alerts through Slack, PagerDuty, email, or webhooks when something looks wrong.

The platform processes roughly 4–6 billion events per day, with traffic spikes most commonly between 2–4 AM and 6–8 AM UTC (aligned with customer ETL jobs — scheduled data processing runs).

---

## First-Day Checklist

Complete these on Day 1, roughly in order:

- [ ] **Datadog**: Ask your manager to add you to the Datadog org. Datadog is the primary tool for logs and application monitoring. Do this on Day 1 — it takes time to provision and you'll need it quickly.
- [ ] **Confluence SSO**: File an IT ticket at `it-help.meridian.internal` (this site does not require SSO — you can access it from a browser immediately). IT provisioning typically takes 2–3 business days. Confluence is where RFCs, runbooks, and internal process docs live.
- [ ] **VPN access**: Several internal tools (Grafana, ArgoCD, Vault) require VPN. Confirm with your manager that VPN is provisioned on your machine before you start dev setup.
- [ ] **GitHub access**: Confirm you're in the Meridian GitHub org and can access the core backend monorepo. If your work touches the frontend, also confirm access to `meridian/frontend`.
- [ ] **Slack channels**: Join these channels (search by name in Slack):
  - `#eng-general` — general announcements and engineering discussion
  - `#new-hires` — your home base for any question, no matter how basic
  - `#code-review` — where you post review requests (not DMs)
  - `#incidents` — follow this to understand how production issues are handled
  - `#platform-rfc` — process questions, RFC eligibility checks
  - `#data-access` — request database access here
  - `#core-domain-reviews` — core domain PR reviews
- [ ] **Dev environment**: Run `make dev` from the core-backend repo root to start your local stack (see Dev Setup below).
- [ ] **Jira access**: Verify you can see the MER project. Branch names require ticket numbers.

---

## Dev Setup

### Prerequisites

- Docker Desktop installed and running
- The `core-backend` repo cloned locally
- VPN access provisioned. Several internal tools require VPN. If `make dev` cannot fetch or inject the dev Vault token, connect to VPN and retry.

### Starting the local stack

```bash
# From the core-backend repo root
make dev
```

**Use `make dev`, not `docker-compose up` directly.** `make dev` wraps docker-compose and also: injects your dev Vault token so services can pull secrets (without it, services fail on startup), configures local Kafka with the correct topic structure, and seeds test customer data so the UI works end-to-end.

Running `docker-compose up` directly will start containers but services will crash at startup because they cannot reach Vault. Tom Weston's approach of running `docker-compose up -d` in a single service directory works once you're familiar with the system and focused on one service, but for your first weeks, use `make dev`. _(Source: Andy K. and Tom Weston in #eng-questions, Feb 28, 2026.)_

**First run**: Image pulls make the first run take several minutes. Subsequent runs are much faster.

### Running tests

```bash
# Fast unit tests (run these frequently)
./gradlew test

# Integration tests (slower — hit real database and Kafka via Testcontainers)
./gradlew integrationTest
```

Integration tests are tagged `@IntegrationTest`. They don't run with the default `test` task because they're slow, but CI runs them on every PR. For examples, look at `processor-svc/src/test/java/io/meridian/processor/integration/` and the `BaseIntegrationTest.java` setup class. _(Source: Andy K. in #eng-questions, Apr 2, 2026.)_

Also required for Kafka schema changes: **contract tests** using Spring Cloud Contract. If you're touching Avro schemas (the message format Kafka uses) in `shared/`, you need contract tests. These ensure that your schema changes don't break services that depend on the message format.

### Coverage requirement

**85% line coverage per module is a hard CI gate** — your PR will fail if coverage drops below this threshold. JaCoCo (the Java code coverage tool) enforces this. The CI report shows which lines are uncovered.

> **Note**: The team-conventions doc describes this as a "target." That wording is outdated. Leila Mohr confirmed on April 2, 2026 (#eng-questions) that JaCoCo threshold enforcement was enabled last quarter (announced March 10 in #eng-general). You must write tests to pass CI. _(Resolved contradiction: team-conventions vs. Slack, Apr 2. Leila Mohr is authoritative as the person who announced and confirmed enforcement.)_

---

## System Overview

Events flow through Apache Kafka (a distributed message broker) as the coordination layer between services. Six backend services, one frontend. Here is what each one does.

### ingestion-api — the event entry point

A REST service that receives incoming events from customers (via a single POST endpoint), validates the auth token and basic structure, and puts each event onto the Kafka message queue (topic `events.raw`) for processing.

> **Resolved: collector-svc is fully decommissioned.** The architecture notes (Dec 2025) described `collector-svc` as an older ingestion surface with 80% of volume migrated to `ingestion-api`. This is out of date. Leila Mohr confirmed in #eng-questions on March 14, 2026 that collector-svc was **fully decommissioned in January 2026** — the last customer migration completed during the holiday week. `ingestion-api` is now the only event entry point. _(Resolved contradiction: arch-notes vs. Slack Mar 14. Leila Mohr's correction is more recent and authoritative; Tom Weston accepted the correction in the same thread.)_
>
> For debugging ingestion issues: use the "Ingestion API - Production" dashboard in Datadog, or `grafana.meridian.internal/d/ingestion-overview` (VPN required).

### processor-svc — the core processing engine

Reads events from `events.raw` and applies the business logic:

- Validates each event field against the customer's registered schema
- Attaches customer metadata (subscription tier, timezone)
- Tracks dataset freshness — whether data arrived within the customer's configured time window
- Detects volume anomalies by comparing current event rate against a 14-day rolling average
- Publishes validated events to `events.validated` (note: the only consumer noted in the source documents is listed as "analytics, future" — confirm current consumers with the Platform team before relying on this topic in active workflows)
- Publishes alert signals to `alerts.triggered`

This is the most critical service. It has a P99 latency SLA of 800ms from event receipt to alert trigger. (P99 means the slowest 1% of requests must still complete within this bound.) When debugging performance issues, start here.

`processor-svc` is the only service that writes to the `events`, `datasets`, and `alert_signals` database tables.

### query-api — read-only data access

Powers the customer-facing dashboard and the public API (`docs.meridian.io`). All reads, no writes. Queries the PostgreSQL read replica.

Most queries are fast and time-bounded. One exception: dataset lineage queries (showing how datasets relate to each other) get slow when customers have complex pipeline graphs. These go through a dedicated slower endpoint with its own connection pool. A ticket (MER-3421) tracks moving lineage to a graph database, but it is not on the H1 roadmap.

The external API uses API key auth. The internal dashboard uses session JWT (JSON Web Token — a signed login/session token).

### scheduler-svc — time-based rule evaluation

Runs periodic jobs to evaluate time-based rules: "this dataset should have received an event in the last 6 hours," end-of-window volume checks, daily report generation, and pruning old alert records.

> **Resolved: scheduler-svc no longer uses Quartz.** The architecture notes describe a Quartz 2.3.2 setup with a database-backed job store. This is out of date. Tom Weston confirmed in #eng-questions on April 18, 2026 that scheduler-svc **migrated from Quartz to Spring `@Scheduled` with Redis distributed locking** in March 2026. If you need to add a scheduled job, look at the existing `@ScheduledJob` beans in `scheduler-svc` — do not follow the Quartz docs. The Redis locking implementation is documented in `RedisJobLock.java` (javadoc). _(Resolved contradiction: arch-notes vs. Slack Apr 18. Tom Weston owns scheduler-svc and self-reports the migration.)_

### notification-svc — alert delivery

Reads alert signals from `alerts.triggered` and delivers them to customer channels: Slack webhook, PagerDuty, email (via SendGrid), or outbound webhook.

For each alert it: checks for duplicate delivery within the customer's deduplication window (default 15 minutes), delivers to all configured channels, records delivery status, and retries on transient failures with exponential backoff for up to 3 hours. Undelivered alerts after 3 hours are marked failed and a support event is created.

### admin-api — internal operations tooling

Internal-only. Not customer-facing. Used by the support and ops team for account management, manual alert suppression, schema debugging, and pipeline diagnostics. Auth is via company SSO (Okta) — separate from customer auth.

Not covered by the public API SLA. Expect occasional planned maintenance.

### frontend

React 18 / TypeScript single-page app, served from GCS (Google Cloud Storage) via a CDN (content delivery network). Talks only to `query-api` (reads) and `ingestion-api` (onboarding wizard). Lives in a separate repo: `meridian/frontend`.

---

## Infrastructure at a Glance

| Component | What it is |
|---|---|
| **Kubernetes / GKE** | Container orchestration. Three clusters: `prod` (production), `staging` (pre-prod), `dev` (personal dev namespaces). |
| **ArgoCD** | Deployment management tool. Merges to `main` auto-deploy to staging. Production is manual with an approval gate. Access: `argocd.meridian.internal` (VPN required). |
| **Kafka (Confluent Cloud)** | Distributed message broker connecting services. Three main topics: `events.raw`, `events.validated`, `alerts.triggered`. |
| **PostgreSQL (CloudSQL)** | Primary database. One primary + one read replica per region. Tenant data is isolated using PostgreSQL schemas (namespaces within a single database) named `tenant_<customer_id>`. |
| **PgBouncer** | Connection pooler in front of PostgreSQL. Reduces database connection overhead — you don't interact with it directly. |
| **HashiCorp Vault** | Secret management. Services pull credentials at startup. Your dev token comes from `make dev`. |
| **Datadog** | Logs, metrics, and distributed tracing. Ask manager to add you Day 1. |
| **Grafana** | Infrastructure dashboards. `grafana.meridian.internal` (VPN required). |
| **Confluent Schema Registry** | Enforces Avro message format compatibility on Kafka topics. New or changed schemas must pass a compatibility check before they can be used. |

> **Flag — Istio service mesh status unknown.** The architecture notes state Istio (a tool for service-to-service encryption and traffic management) was planned for Q1 2026. Q1 2026 has passed, but none of the sources from Feb–Apr 2026 mention whether it was deployed. Without Istio, inter-service communication uses internal Kubernetes DNS without mutual TLS (encryption between services). The current state is unverifiable from these documents — check with the Platform team or Priya Nair before assuming whether mTLS is enforced. _(Reason for uncertainty: arch-notes are the only source on Istio; Slack threads from the same period never mention it, which is weak evidence either way.)_

> **Flag — PostgreSQL version unconfirmed.** The architecture notes specify PostgreSQL 13. This is from December 2025 and CloudSQL instances can be upgraded. No 2026 source confirms the current version. If version matters for something you're working on (e.g., relying on a version-specific feature), confirm with the Platform team. _(Reason: no source more recent than Dec 2025 mentions the PG version.)_

---

## Data Model Basics

PostgreSQL uses a **multi-tenant schema** design. Each customer gets their own PostgreSQL schema (a namespace within a single shared database, not a separate database) named `tenant_<customer_id>`. Shared reference data lives in the `public` schema.

Key tables:
- `public.customers` — account records, subscription tier, feature flags
- `public.schema_registry` — event schemas registered by customers
- `tenant_<id>.events` — validated events, partitioned by day, retained 90 days
- `tenant_<id>.datasets` — dataset metadata and current status
- `tenant_<id>.alert_signals` — alert history and delivery state

Database access is role-based. Request access in `#data-access` — tag Leila Mohr as approver. Credentials come from Vault once approved. Typical turnaround: 1–2 business days. Request access when you need it for a specific ticket, not on Day 1 universally.

---

## Key People

The table below reflects what the source documents actually show about each person's responsibilities, not assumed org-chart titles (which may differ from HR records).

| Person | What the sources show | When to involve them |
|---|---|---|
| **Priya Nair** | Lead Architect; structural core domain owner; maintains the architecture notes | Structural changes to the core domain model; major architectural decisions; architecture questions in `#platform-rfc` |
| **Tom Weston** | Day-to-day `core-domain/` reviewer; scheduler-svc contact; took over reviews when Priya moved to architecture role | Normal `core-domain/` PRs, new domain events within existing aggregates, scheduler-svc questions; tag in PRs and `#core-domain-reviews` |
| **Andy K.** | Author of team-conventions.md; frequent first responder to new-hire questions in Slack | Local dev setup, RFC template access, process questions; ping in `#new-hires` or `#eng-general` |
| **Leila Mohr** | DB access approver; platform/process authority (confirms coverage gate enforcement, RFC scope, migration processes) | Tag in `#data-access` requests; ask about RFC eligibility or platform-wide process questions |
| **On-call engineer** | Rotating production approver | Production deployment approvals; hotfix sign-off; see `#incidents` for current rotation |

> **Resolved: core-domain review ownership has split.** The team-conventions doc says to loop in Priya for all core-domain changes. This is partially outdated. Andy K. and Tom Weston confirmed in #eng-questions on March 22, 2026 that **Tom Weston now handles day-to-day core-domain PR reviews** since Priya moved to a full architecture role. For a new domain event within an existing aggregate, ping Tom directly and tag him in `#core-domain-reviews`. For structural changes to the domain model itself, Tom will involve Priya. _(Resolved contradiction: team-conventions vs. Slack Mar 22. Both Andy K. and Tom Weston confirm the split; Tom self-reports his own scope.)_

---

## How Work Gets Done

### Branching and commits

Branch names follow `initials/MER-<ticket>-slug` format:
```
jn/MER-1234-add-freshness-endpoint
```

A ticket number is required. If there's no ticket, create one. You'll also see branches using `username/ticket-slug` (e.g., `andyk/MER-456-refactor`) — both formats appear in the history. Commit messages should explain what changed and why — "fix" or "wip" are not acceptable. Most people use conventional commits format (e.g., `feat:`, `fix:`, `refactor:`).

When a PR merges, **squash merge** is the default — your branch's individual commits get collapsed into one commit on `main`. This is expected behavior.

### Pull requests

- **Two approvals required** for all PRs. One approval is acceptable for doc changes, config tweaks, and dependency bumps that don't affect runtime behavior — use judgment.
- Open as a draft early if you want feedback on your approach. Only request reviews when it's ready to merge.
- Expect a 24-hour first-pass turnaround. If you're on a deadline, say so in the PR description. Post review requests in `#code-review`, not DMs.
- **Do not merge on Fridays.** On-call is lighter on weekends. This is cultural, not a hard block — but you will hear about it if you merge something risky on a Friday afternoon.

### Deploying

Merging to `main` automatically triggers a staging deploy via ArgoCD. If staging breaks, you are responsible for fixing or rolling back before others merge.

Production deploys are manual: open ArgoCD (`argocd.meridian.internal`, VPN required), find your service, click Sync. This triggers an approval gate — you need a thumbs-up from your team lead or the on-call engineer. Deploy during business hours (9 AM–5 PM PT); avoid Fridays.

Rollbacks are fast in ArgoCD: point to the previous image tag and sync. For any user-facing impact: roll back first, investigate after.

### When you need an RFC

File an RFC **before you write code** for any:
- Change to the external API (new endpoints, changed request/response shapes) — even adding an optional field counts, because some customers parse responses strictly
- New Kafka message schema or changes to an existing one

RFC template: Confluence → Engineering → RFCs → Template. Mention relevant team leads in the RFC. Review SLA is 5 business days. For eligibility questions, ping `#platform-rfc` — someone usually responds within 15 minutes.

**If your Confluence SSO isn't ready yet**: Ask Andy K. or your team lead for the template directly. Leila Mohr and Andy K. gave this workaround in the #eng-questions thread on March 5, 2026.

New Kafka **topics** are separate from RFCs. A new topic requires a Terraform PR in `platform-infra` (not an RFC). If a new topic is part of a cross-service API change, file both in parallel, not sequentially.

### Core domain changes

If you're adding a new entity, changing a domain event, or modifying anything in `core-domain/`:

1. Ping Tom Weston directly (or tag in `#core-domain-reviews`) — he handles day-to-day reviews
2. Tom will loop in Priya if the change is structural to the domain model

Plan for a conversation, not just a PR comment, for anything touching domain event structure.

### Schema migrations

Database migrations use Flyway (the tool used to apply and track database schema changes in a controlled way). **Destructive migrations** (dropping columns, changing constraints) require a review from the Core team and a documented rollback plan. This is a hard requirement, not a suggestion.

---

## Tools Reference

| Tool | URL / Access | Notes |
|---|---|---|
| Datadog | Via browser (ask manager for org access on Day 1) | APM, logs, distributed tracing |
| Grafana | `grafana.meridian.internal` | VPN required. Infra dashboards |
| ArgoCD | `argocd.meridian.internal` | VPN required. Deployments |
| HashiCorp Vault | `vault.meridian.internal` | VPN required. Dev token from `make dev` |
| Confluence | Via SSO (provision at `it-help.meridian.internal`) | Docs, RFCs, runbooks. Takes 2–3 days to provision |
| IT Help | `it-help.meridian.internal` | No SSO needed. Use on Day 1 to file SSO ticket |
| Jira | Via SSO | Issue tracking. All branches need a MER- ticket number |

---

## Getting Help

- **Stuck on code?** Post in `#eng-general` or DM whoever last touched the area (`git log` on the relevant files is your friend).
- **Process questions?** Post in `#new-hires`. No question is too basic.
- **Architecture questions?** Post in `#platform-rfc` or ping Priya or your team lead directly.
- **Stuck for 30+ minutes?** Ask. Don't stay blocked.

---

## Appendix: Resolved Contradictions

| Topic | Source A (older) | Source B (newer/corrective) | Resolution |
|---|---|---|---|
| `collector-svc` status | arch-notes (Dec 2025): 80% migrated, still running | Slack (Mar 14, 2026, Leila Mohr): fully decommissioned Jan 2026; Tom Weston accepts correction | **Decommissioned.** Leila Mohr's March 2026 correction is authoritative. |
| scheduler-svc technology | arch-notes (Dec 2025): Quartz 2.3.2, database-backed | Slack (Apr 18, 2026, Tom Weston): Spring @Scheduled + Redis locking since March 2026 | **Quartz is gone.** Tom Weston owns scheduler-svc and self-reports the migration. |
| core-domain reviewer | team-conventions (Mar 2026): always Priya | Slack (Mar 22, 2026, Andy K. + Tom Weston): Tom for day-to-day, Priya for structural | **Split ownership.** Both parties confirm the new arrangement. |
| 85% coverage enforcement | team-conventions: "target" | Slack (Apr 2, 2026, Leila Mohr): hard CI gate, enforced since Mar 10 announcement | **Hard gate.** Leila Mohr announced enforcement; Dan Reyes retracts his contrary claim in the same thread. |
| `make dev` vs `docker-compose up` | README: both options mentioned | Slack (Feb 28, 2026, Andy K.): `make dev` is correct for new hires; docker-compose alone fails without Vault setup | **Use `make dev`.** Andy K. is explicit; Tom Weston confirms the exception is for experienced engineers on single services. |

## Appendix: Flagged Uncertain Claims

| Claim | Why uncertain |
|---|---|
| Istio service mesh deployment | arch-notes (Dec 2025) describe Istio as planned for Q1 2026. Q1 has passed. No Feb–Apr 2026 Slack thread mentions Istio. Cannot verify whether it was deployed. This matters because it determines whether mTLS is enforced between services. |
| PostgreSQL version | arch-notes (Dec 2025) say PostgreSQL 13. No 2026 source confirms this. CloudSQL instances can be upgraded in place. Matters if you're working on features or bugs specific to a PG version. |
| IT auto-creates SSO ticket on hire | Slack (Leila Mohr, Mar 5): "IT should have auto-created an SSO ticket when you started." The word "should" signals this is not guaranteed. File one manually at `it-help.meridian.internal` on Day 1 regardless. |
| `events.validated` consumers | arch-notes list the consumer as "(analytics, future)." No 2026 source clarifies whether any analytics consumer is now active. Matters if you're building something that expects to consume this topic. |
