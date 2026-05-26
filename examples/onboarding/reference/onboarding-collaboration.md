# First-Week Guide: Engineering at Meridian Systems

_Synthesized from three sources: architecture notes (arch-notes.md, last updated Dec 2025), team conventions (team-conventions.md, last updated Mar 2026), and #eng-questions Slack threads (Feb–Apr 2026). Where sources disagreed, the more recent and more authoritative source was preferred — contradictions are documented explicitly at the bottom._

---

## What Meridian Does

Meridian is a **data observability platform**. Customers connect their data pipelines by sending events to Meridian's ingestion endpoint. Meridian then:

1. Validates those events against the customer's registered schemas
2. Detects anomalies (volume spikes, drops) and freshness violations (data arriving late or not at all)
3. Delivers alerts through the customer's chosen channels: Slack, PagerDuty, email, or webhook

The platform handles **4–6 billion events per day** at a peak of ~80,000 events/second. That scale drives some carefully tuned performance requirements — the most critical service (processor-svc) has an 800ms P99 latency SLA from event receipt to alert trigger.

---

## Key People

| Name | Role | Go to them for |
|---|---|---|
| **Priya Nair** | Lead Architect | Structural changes to the domain model, architecture decisions, design reviews |
| **Andy K.** | Senior Engineer | Dev setup, process questions, RFC templates — first point of contact for new hires |
| **Leila Mohr** | Team Lead / Platform | RFC eligibility, DB access approvals, external API changes, CI/CD policy |
| **Tom Weston** | Engineer / Core Domain | Day-to-day core-domain PR reviews, scheduler-svc questions |

**How to reach people:** Slack is the primary channel. DMs are fine for quick questions; use public channels for things others might benefit from seeing.

---

## Day One Checklist

Work through this in order — some items unblock others. Slack comes first because it's how you resolve every other blocker.

- [ ] **Join Slack channels** — listed below. The `#new-hires` channel is specifically for new hire questions; no question is too basic.
- [ ] **Get added to Datadog** — ask your manager on day one. Datadog is how you investigate most issues (APM, logs, distributed tracing). You'll need this before your first debugging session.
- [ ] **Check your Confluence / SSO access** — try logging in. If Confluence blocks you, file an IT ticket immediately at `it-help.meridian.internal` (this site does not require SSO). Provisioning takes 2–3 business days. Don't wait — Confluence has RFC templates and runbooks.
- [ ] **Confirm VPN access** — Grafana and ArgoCD require VPN to access. Ask your manager whether VPN was set up as part of your IT onboarding.
- [ ] **Clone the repositories** — `meridian/core-backend` (main backend monorepo) and `meridian/frontend` (React SPA, separate repo).
- [ ] **Run `make dev`** in core-backend — this starts the full local stack. First run is slow (image pulls); subsequent runs are fast.
- [ ] **Introduce yourself** — post in `#new-hires` or `#eng-general`.

### Slack Channels to Join

| Channel | What it's for |
|---|---|
| `#eng-general` | General engineering discussion, announcements |
| `#code-review` | Review requests (preferred over DMs — someone else might be free) |
| `#incidents` | Active incidents, postmortems, hotfix documentation |
| `#platform-rfc` | RFC eligibility checks, RFC discussions |
| `#data-access` | Database access requests |
| `#new-hires` | New hire questions — no question too basic |
| `#core-domain-reviews` | Non-urgent core-domain PR review requests |

---

## Development Environment Setup

### Prerequisites

- **Docker + Docker Compose** — required for `make dev`
- **VPN** — required for Grafana and ArgoCD (both at `*.meridian.internal` URLs); see Day One checklist
- **Java toolchain** — backend services are Spring Boot; check the core-backend README for the required JDK version

### Step 1: Clone the repos

```
git clone git@github.com:meridian/core-backend.git
git clone git@github.com:meridian/frontend.git
```

`core-backend` is the main monorepo containing all backend services. `frontend` is a separate repo with the React 18/TypeScript single-page app. If you're a backend engineer, you likely won't need the frontend repo initially.

### Step 2: Start the local stack

From the `core-backend` repo root:

```bash
make dev
```

**Use `make dev`, not `docker-compose up` directly.** Running `docker-compose up` alone will start the containers, but services will fail at startup because they can't pull secrets from Vault. `make dev` wraps docker-compose and additionally:

- Injects a dev Vault token so services can pull secrets
- Configures local Kafka (the message broker services use to pass events asynchronously) with the correct topic setup
- Seeds test customer data so the UI works end to end

If you only need one service and you already know what you're doing, `docker-compose up -d` in a service directory can work — but you lose the test data seeding. **For your first week, always use `make dev` from the repo root.**

**First run is slow** — it downloads many container images. Plan for 10–20 minutes. Subsequent runs start in seconds.

### Step 3: Get Datadog access (day one)

Ask your manager to add you to the Datadog org. Datadog is your primary tool for:
- APM and distributed tracing
- Centralized logs across all services
- Production dashboards (e.g., "Ingestion API - Production")

### Step 4: Get access to VPN-gated tools

Once you have VPN, these become accessible:

| Tool | URL | What it's for |
|---|---|---|
| Grafana | `grafana.meridian.internal` | Infrastructure dashboards |
| ArgoCD | `argocd.meridian.internal` | Deployment UI — promote to staging/prod |
| Vault | `vault.meridian.internal` | Secrets (your dev token comes automatically from `make dev`) |

### Step 5: Confluence access

Confluence is where RFCs, runbooks, and internal docs live. It requires SSO (not VPN). If Confluence blocks you when you try to log in, file a ticket at `it-help.meridian.internal` — that site doesn't require SSO either. Takes 2–3 business days. In the meantime, ask Andy or your team lead for any templates you need urgently.

### Step 6: Database access (when you need it)

You won't need this on day one. When it comes up: post in `#data-access` with your name, team, what you need (read replica or specific tenant schemas), and the Jira ticket. Tag **@Leila Mohr** as the approver. Credentials come from Vault. Provisioning takes **1–2 business days**.

---

## System Architecture (What You Need in Week 1)

You don't need to understand every detail before writing your first PR. Here's what matters week one.

### The six backend services

Events flow through the system in this order:

```
Customer → ingestion-api → [Kafka: events.raw] → processor-svc
                                                      ↓
                                           [Kafka: alerts.triggered]
                                                      ↓
                                           notification-svc → Customer channels
```

Apache Kafka is the distributed message broker that services use to pass events asynchronously — think of it as a durable queue that services write to and read from independently.

| Service | What it does | Notes |
|---|---|---|
| **ingestion-api** | Receives events from customers via REST POST. Validates auth and basic structure. Publishes to Kafka. | The only event entry point. Stateless Spring Boot, scales horizontally. |
| **processor-svc** | Core logic: schema validation, enrichment, freshness tracking, anomaly detection. | Most critical service. 800ms P99 SLA. Owns the `events`, `datasets`, `alert_signals` tables. |
| **scheduler-svc** | Time-based freshness rules, periodic jobs, daily reports, alert pruning. | Uses Spring `@Scheduled` (Spring's built-in annotation for periodic jobs) with Redis distributed locking. |
| **notification-svc** | Delivers alerts to customer channels. Deduplicates. Retries with exponential backoff for up to 3 hours. | Eventually consistent — a delivery failure is retried, not dropped. |
| **query-api** | All reads for the dashboard and external API. Queries the PostgreSQL read replica. | No writes. Powers both customer-facing dashboard and external API. |
| **admin-api** | Internal ops tooling for the support team. Auth via Okta SSO. | Not customer-facing. Not covered by the public SLA. |

The **frontend** (React 18/TypeScript) lives in a separate repo (`meridian/frontend`) and talks only to `query-api` (reads) and `ingestion-api` (onboarding wizard). No direct database or Kafka access.

### The three Kafka topics

| Topic | Produced by | Consumed by | Retention |
|---|---|---|---|
| `events.raw` | ingestion-api | processor-svc | 24 hours |
| `events.validated` | processor-svc | (analytics, future) | 7 days |
| `alerts.triggered` | processor-svc, scheduler-svc | notification-svc | 72 hours |

All Kafka messages use Avro — a compact binary serialization format with a schema. Schema changes require a compatibility check through Confluent Schema Registry before they can be published.

### The database

The database is PostgreSQL on CloudSQL (see "Things We Couldn't Confirm" for a note on the version). Per-tenant isolation uses PostgreSQL schemas — these are namespaces within a single database, not separate databases. Each customer gets a schema named `tenant_<customer_id>`. Shared reference data lives in the `public` schema.

### Infrastructure at a glance

- **Kubernetes on GKE**, three clusters: `dev`, `staging`, `prod`
- **CI/CD**: GitHub Actions for build/test; ArgoCD for deployment
- **Secrets**: HashiCorp Vault — never hardcode secrets, never use Kubernetes Secrets for sensitive values
- **Observability**: Datadog (APM, logs), Grafana (infra metrics, VPN required)

---

## Working Conventions

### Branches

Format: `initials/MER-XXXX-short-description`

```
jd/MER-1234-fix-auth-token-refresh
```

A Jira ticket is required for every branch. If there isn't one, create it. Stale branches are pruned monthly. (You'll also see the older `fullname/ticket-slug` format in repo history — both are valid.)

### Pull Requests

- **Two approvals** required for all PRs. One is acceptable for docs, config tweaks, and dependency bumps that don't touch runtime behavior — when in doubt, get two.
- Open as **draft** early for approach feedback before it's ready to merge.
- Expect a **24-hour first-pass review**. If you're blocked, ping `#code-review` (not DMs — someone else might pick it up).
- PRs are **squash-merged** by default — your commit history will be collapsed into one commit when merged.
- **Don't deploy on Fridays.** On-call is lighter on weekends and rollbacks are harder to coordinate. This is cultural, but you will hear about it.

### Tests and Coverage

**85% line coverage per module is a hard CI gate**, enforced by JaCoCo. Your PR will fail CI if it drops below 85%. (The conventions doc says "target" — that wording is outdated. It is enforced. See Contradictions section.)

Three test types:

| Type | Command | Notes |
|---|---|---|
| Unit tests | `./gradlew test` | Fast. Mocks everything external. Run these constantly. |
| Integration tests (`@IntegrationTest`) | `./gradlew integrationTest` | Slow. Uses Testcontainers (a library that starts real Postgres and Kafka instances in Docker) to exercise real database/broker behavior. CI runs on every PR; excluded from the default test task. |
| Contract tests | (part of CI) | Required for any Avro schema change in `shared/`. Uses Spring Cloud Contract. |

For integration test examples: `processor-svc/src/test/java/io/meridian/processor/integration/`. Base class: `BaseIntegrationTest.java`.

### Deployments

Merge to `main` → automatic ArgoCD sync to **staging**. No approval. If you break staging, fix it before anyone else merges.

**Production** is manual:
1. Open ArgoCD (`argocd.meridian.internal`, VPN required)
2. Find your service, click Sync
3. Approval gate requires a thumbs-up from your team lead or the on-call engineer

Deploy during business hours (9 AM–5 PM PT). Avoid Fridays.

For rollbacks: point ArgoCD to the previous image tag and sync. If there's user-facing impact, roll back first and investigate after.

### RFC vs Design Review — Know the Difference

This is easy to confuse. The short version:

| Situation | What you need |
|---|---|
| Adding or changing a customer-facing API endpoint | RFC |
| Adding an optional field to an API response | RFC (even optional fields — customers parse responses strictly) |
| New Kafka message schema or cross-service contract | RFC |
| Creating a new Kafka **topic** | Design Review + Terraform PR in `platform-infra` |
| Both at once (new topic + new schema for external contract) | RFC and Terraform PR in parallel |

RFC template: Confluence → Engineering → RFCs → Template. Mention relevant team leads when filing. Review SLA: 5 business days. Not sure if you need an RFC? Post in `#platform-rfc` — answer in ~15 minutes.

### core-domain Changes

For a new domain event or routine core-domain work: ping **Tom Weston** directly and tag him in `#core-domain-reviews`. He handles day-to-day reviews.

For structural changes to the domain model (new entities, changes to how domain events are structured): ping **Priya Nair** — plan for a conversation, not just a PR comment.

---

## Contradictions Between Sources — How They Were Resolved

Three documents fed into this guide, written between December 2025 and April 2026. Where they disagreed, here is which source was trusted, and why.

### 1. Is collector-svc still running?

| Source | Says |
|---|---|
| arch-notes.md (Dec 2025) | ~80% migrated; legacy customers remain on collector-svc in maintenance mode |
| slack-qa.md Thread 4 (Mar 2026) | collector-svc was fully decommissioned in January 2026; all customers are on ingestion-api |

**Trusted: the Slack thread.** Leila Mohr (team lead) explicitly corrected Tom's assumption in real time, and Tom accepted the correction. The arch notes carry their own disclaimer: "These notes are a working reference, not a specification." The Slack thread is three months more recent.

**Secondary consequence:** the arch-notes Kafka topic table lists `events.raw` as produced by `ingestion-api, collector-svc`. With collector-svc decommissioned, this is also stale — only `ingestion-api` produces to `events.raw`. The Kafka topic table in this guide already reflects the current state.

**What this means for you:** ingestion-api is the only event entry point. Use the "Ingestion API - Production" Datadog dashboard when debugging ingestion flows.

### 2. What does scheduler-svc use for job scheduling?

| Source | Says |
|---|---|
| arch-notes.md (Dec 2025) | Quartz 2.3.2 with a database-backed job store |
| slack-qa.md Thread 8 (Apr 2026) | Migrated to Spring `@Scheduled` + Redis distributed locking in March 2026 |

**Trusted: the Slack thread.** Tom Weston, who owns scheduler-svc, directly confirmed the migration while answering a question about it. The code is your primary reference: look for `@ScheduledJob` beans; `RedisJobLock.java` has a detailed javadoc explaining the locking approach. Lock key is `scheduler:<job-name>`; TTL is configurable per job via the `@ScheduledJob` annotation.

### 3. Who reviews core-domain changes?

| Source | Says |
|---|---|
| team-conventions.md (Mar 2026) | "Changes to core-domain/ need a review from Priya" |
| slack-qa.md Thread 5 (Mar 2026) | Tom handles day-to-day reviews; Priya's involved for structural domain model changes |

**Trusted: the Slack thread — refines rather than contradicts.** Both Andy and Tom independently confirmed the split in the same thread. The conventions doc isn't wrong; it just describes Priya's role without capturing the current division of labor. For routine work (adding a domain event), start with Tom. Tom will pull in Priya if needed.

### 4. Is 85% test coverage a target or a hard gate?

| Source | Says |
|---|---|
| team-conventions.md | "We target 85% line coverage" — implies aspirational |
| slack-qa.md Thread 6 (Apr 2026) | Leila confirmed 85% minimum is enforced by CI; announced March 10 in #eng-general |

**Trusted: the Slack thread.** Leila explicitly corrected the "it's just a target" interpretation, referencing the March 10 announcement. Jamie's PR actually failed CI at 83.4%, confirming the gate exists. The doc wording is a holdover from before enforcement was enabled. **85% is a hard gate; your PR will not merge below it.**

---

## Things We Couldn't Confirm — And Why

These are genuine gaps where no source document has a clear, current answer. They're flagged with the reason so you can judge how much it matters for your work right now.

### 1. PostgreSQL version

The December 2025 architecture notes say PostgreSQL 13 on CloudSQL. No Slack thread or other document confirms the version in 2026. Given that two other claims from those same arch-notes (collector-svc status, Quartz) are confirmed stale, the PG13 version has the same staleness risk.

**Why this matters:** if you're writing migrations or using version-specific PostgreSQL features, you want to know the actual version.

**How to find out:** check `platform-infra` (Terraform has the CloudSQL config), or ask in `#eng-general`.

_(The per-tenant schema structure — `tenant_<customer_id>` namespaces — is corroborated by the #data-access Slack thread and is not version-dependent, so that part is trustworthy.)_

### 2. Whether Istio is deployed (was planned for Q1 2026)

The architecture notes say Istio service mesh was planned for Q1 2026. Today is May 2026 — the deadline has passed, and no other document confirms whether it shipped.

**Why this matters:** if Istio is live, mTLS between services may now be enforced. If it isn't, the previous state (cluster-internal DNS, no enforced mTLS) still applies. This affects inter-service communication design.

**How to find out:** ask in `#eng-general`, or check `platform-infra` for Terraform/Helm changes related to Istio.

### 3. Redis as platform infrastructure

scheduler-svc migrated to Redis for distributed locking, but the architecture notes (from before the migration) don't mention Redis. It's unclear whether Redis is a new managed service, a GKE add-on, or something else.

**Why this matters:** if you're writing a new scheduled job or debugging a locking issue, you need to know what Redis instance you're working with.

**How to find out:** ask Tom Weston, or check `platform-infra` for Redis resources.

### 4. VPN setup process

Grafana and ArgoCD require VPN. None of the source documents explain how to get or configure VPN access. It's reasonable to assume this is part of IT onboarding, but the process isn't documented here.

**Why this matters:** without VPN you can't access Grafana or ArgoCD from outside the office. You'll need ArgoCD to do your first production deployment.

**How to find out:** ask your manager whether VPN was included in your IT setup on day one.

### 5. Frontend local development setup

The architecture notes mention `meridian/frontend` as a React 18/TypeScript SPA, but none of the three source documents describe how to run it locally. The `make dev` command is for the backend stack only.

**Why this matters:** if you're working on the frontend, you need to know how to run it.

**How to find out:** check the README in `meridian/frontend`.

---

_This guide was synthesized in May 2026 from documents spanning December 2025–April 2026. The Contradictions section exists specifically so that when you find something that doesn't match, you understand the original reasoning and can judge what to trust. If you find an error or something outdated, post in `#new-hires`._
