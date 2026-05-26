# First-Week Guide — Meridian Systems Engineering

_Compiled from arch-notes.md (Priya Nair, Dec 2025), team-conventions.md (Andy K., Mar 2026), and #eng-questions Slack threads (Feb–Apr 2026). Where those sources conflict, the more recent Slack thread wins — it reflects what the team actually does today, not what was written months ago. Conflicts and unresolved items are called out explicitly below._

---

## What You're Joining

Meridian is a data observability platform. Customers connect their data pipelines by sending events to Meridian; Meridian validates those events against customer-defined schemas and freshness rules, detects anomalies, and delivers alerts through Slack, PagerDuty, email, or webhooks.

In production, the platform processes 4–6 billion events per day, with peak bursts of ~80,000 events/second (mostly 2–4 AM and 6–8 AM UTC, aligned with customer ETL windows).

---

## Key People

| Person | Role | Go to them for |
|--------|------|----------------|
| **Priya Nair** | Lead Architect | Structural changes to the domain model; architecture decisions; PostgreSQL access approvals |
| **Tom Weston** | Core Domain Reviewer (day-to-day) | `core-domain/` PR reviews; scheduler-svc questions; anything touching domain events or aggregates |
| **Leila Mohr** | Platform team | Database access approval (tag her in `#data-access`); RFC questions; platform CI/coverage questions |
| **Andy K.** | Engineering conventions | Team conventions questions; RFC template; dev setup questions |
| **On-call engineer** | Weekly rotation | Production deployment approval; hotfix coordination |

> **Note on Priya vs. Tom for core-domain:** The team-conventions doc says all `core-domain/` changes need Priya. That was accurate when it was written, but it's out of date. As of March 2026, Tom handles routine core-domain reviews and Priya is consulted for structural/architectural decisions. When in doubt, ping Tom first — he'll escalate to Priya if needed. (Source: #eng-questions, Mar 22, Tom Weston.)

---

## Day One Checklist

- [ ] Get your laptop set up with Git, Java (check which version in `core-backend` README), Docker Desktop
- [ ] Ask your manager to add you to the **Datadog** org (do this on day one — provisioning takes time)
- [ ] File an **IT SSO ticket** at `it-help.meridian.internal` (no SSO required to access that site) — you'll need SSO for Confluence, ArgoCD, Grafana, and Vault. Expect 2–3 business days.
- [ ] Connect to **VPN** — required for Grafana (`grafana.meridian.internal`) and ArgoCD (`argocd.meridian.internal`)
- [ ] Clone the main repo (`meridian/core-backend`) and run `make dev` to verify your local stack starts (see Dev Environment Setup below)
- [ ] Clone the frontend repo (`meridian/frontend`) if your role touches the UI
- [ ] Join the Slack channels listed in the Channels section below
- [ ] Post a brief intro in `#new-hires` — that's also the right place for any question, no matter how basic
- [ ] Find your first ticket in Jira and make sure it has a ticket number (branch naming requires one)

---

## Dev Environment Setup

### Step 1: Clone the repo

```bash
git clone git@github.com:meridian/core-backend.git
cd core-backend
```

The frontend is a separate repo:
```bash
git clone git@github.com:meridian/frontend.git
```

### Step 2: Start the local stack

```bash
make dev
```

**Use `make dev`, not `docker-compose up` directly.** The `Makefile` does three things that bare `docker-compose` skips:
1. Injects a dev Vault token so services can pull secrets at startup
2. Configures local Kafka with the correct topic layout
3. Seeds test customer data so the UI is usable end-to-end

If you run `docker-compose up` without `make dev`, services will crash on startup because the Vault config is missing. (Advanced: running `docker-compose up` inside a single service directory is fine once you know what you're doing, but start with `make dev`.)

First run is slow — it pulls a lot of images. Subsequent runs are fast.

### Step 3: Verify it works

Once `make dev` finishes:
- The local UI should be accessible (check the README for the port)
- Services log to stdout — watch for startup errors in the first 30 seconds
- If a service fails to start, the most common cause is Vault token issues — re-run `make dev`

### Step 4: Request database access (when needed)

You won't need production database access on day one, but when you do:

1. Post in `#data-access` with: your name, team, whether you need the full read replica or specific tenant schemas, and the Jira ticket number
2. Tag `@Leila Mohr` as approver
3. Wait 1–2 business days — credentials are issued from Vault

Do not request write access to production unless your role explicitly requires it.

### Step 5: Get Confluence access (for RFCs and runbooks)

SSO access is required. File an IT ticket at `it-help.meridian.internal` on day one — it takes 2–3 business days. If you need the RFC template urgently before your SSO is ready, DM Andy K. or your team lead.

---

## How the System Works (What You Need to Know First)

### Event flow

```
Customer → ingestion-api → events.raw (Kafka) → processor-svc → events.validated (Kafka)
                                                              └→ alerts.triggered (Kafka) → notification-svc → customer channels
                                                 scheduler-svc → alerts.triggered (Kafka) ↗
```

- **ingestion-api**: stateless Spring Boot service, accepts events via POST, validates auth, publishes to Kafka. This is the _only_ customer entry point as of January 2026.
- **processor-svc**: most critical service. Validates schemas, enriches events, detects anomalies, manages freshness state. P99 latency SLA is 800ms. Owns the `events`, `datasets`, and `alert_signals` PostgreSQL tables — no other service writes to these.
- **scheduler-svc**: runs periodic jobs for time-based freshness checks and reports. Uses Spring `@Scheduled` with Redis distributed locking (see the "Contradictions Resolved" section below).
- **notification-svc**: delivers alerts via Slack, PagerDuty, email (SendGrid), or webhooks. Retries on failure with exponential backoff up to 3 hours.
- **query-api**: read-only REST API for the customer dashboard and external API. Queries the PostgreSQL read replica.
- **admin-api**: internal support/ops tooling. Okta-authenticated, not customer-facing.

### Where data lives

- **PostgreSQL** (CloudSQL): primary + read replica. Tenant isolation via PostgreSQL schemas (`tenant_<customer_id>`).
- **Kafka** (Confluent Cloud): three topics — `events.raw` (24h retention), `events.validated` (7d), `alerts.triggered` (72h). All messages use Avro; schema changes go through the Confluent Schema Registry.

### Dashboards

- **Datadog** — "Ingestion API - Production" for ingestion metrics; APM and logs for all services
- **Grafana** — `grafana.meridian.internal/d/ingestion-overview` for infra view (VPN required)

---

## Day-to-Day Conventions

### Branches

Format: `initials/ticket-slug`

```
js/MER-1601-check-query-patterns
tw/MER-1456-add-dataset-lineage-endpoint
```

Every branch needs a ticket number. If one doesn't exist, create it. Stale branches are pruned monthly.

### Pull requests

- **Two approvals** required (one is okay only for docs, config, and dep bumps that don't touch runtime behavior — use judgment)
- Open as **draft** early for approach feedback; don't request reviews until you're ready to merge
- Expect a **24-hour first-pass** turnaround; if you're blocked, post in `#code-review` (not DMs)
- **No Friday deploys** — this is cultural, not enforced by tooling, but you will hear about it

### Tests and coverage

- Target is **85% line coverage per module**, measured by JaCoCo
- This is a **hard CI gate** — PRs that drop below 85% will fail to build. (The team-conventions doc describes it as a "target," which is misleading — it was enforced by CI as of March 10, 2026. See "Contradictions Resolved" below.)
- Three test levels:
  - **Unit tests**: `./gradlew test` — fast, mocks everything external
  - **Integration tests** (`@IntegrationTest`): `./gradlew integrationTest` — real Postgres/Kafka via Testcontainers. Slow; CI runs them on every PR. Examples in `processor-svc/src/test/java/io/meridian/processor/integration/`; base class is `BaseIntegrationTest.java`
  - **Contract tests**: required for any Avro schema change in `shared/`

### Deployments

- Merge to `main` → **staging auto-deploys** (ArgoCD)
- Production is **manual** — open ArgoCD, find your service, click Sync, get approval from team lead or on-call engineer
- Deploy during **9 AM–5 PM PT**, avoid Fridays
- Rollbacks: ArgoCD, point to previous image tag and sync. Default posture: roll back first, investigate after.

### RFC process

Required before writing code for:
- Any cross-service API change
- Any change to how customers integrate with Meridian (new endpoints, changed request/response shapes, new Kafka schemas)

Even adding an optional field to the external API requires an RFC (some customers parse responses strictly — this came up explicitly in a March 2026 thread).

To start: post in `#platform-rfc` and someone will tell you within ~15 minutes whether you need one. If you do, the template is on Confluence (Engineering → RFCs → Template). Mention the relevant team leads; review SLA is 5 business days.

New Kafka topics are separate: open a Terraform PR in `platform-infra` and tag the Platform team. The PR template covers partition count, retention, and schema registration. This is a Design Review, not an RFC — run them in parallel if both are needed.

### Secrets

Never hardcode secrets. Never use Kubernetes Secrets directly for sensitive values. All secrets come from HashiCorp Vault (`vault.meridian.internal`); your dev token comes from `make dev`.

---

## Slack Channels

| Channel | Purpose |
|---------|---------|
| `#eng-general` | General engineering discussion and announcements |
| `#code-review` | Review requests — preferred over DMs |
| `#incidents` | Active incidents, postmortems, hotfix documentation |
| `#platform-rfc` | RFC eligibility checks and discussions |
| `#data-access` | Database access requests |
| `#new-hires` | Questions from new engineers — no question too basic |
| `#core-domain-reviews` | Non-urgent core-domain PR review requests |

---

## Contradictions Resolved

### 1. Is collector-svc still running?

**Short answer: No.** ingestion-api is the only customer entry point.

- **arch-notes.md** (Dec 2025) says ~80% of event volume is on ingestion-api, with legacy customers still on collector-svc in maintenance mode.
- **Leila Mohr in Slack** (Mar 2026) clarified that collector-svc was fully decommissioned in January 2026 — the last migration batch completed during the holiday week. The arch notes are out of date on this point.

**Trusted source:** Leila Mohr's Slack message (more recent, from someone with direct platform knowledge). Don't look at collector-svc metrics or code for debugging ingestion issues — use "Ingestion API - Production" in Datadog.

### 2. What does scheduler-svc use for job scheduling?

**Short answer: Spring `@Scheduled` with Redis distributed locking** — not Quartz.

- **arch-notes.md** describes Quartz 2.3.2 with a PostgreSQL-backed job store.
- **Tom Weston in Slack** (Apr 2026) says the migration from Quartz happened in March 2026. The service now uses Spring `@Scheduled` with a custom `@ScheduledJob` annotation. The locking implementation is documented in `RedisJobLock.java` (javadoc in the scheduler package).

**Trusted source:** Tom Weston (Mar 2026) — the arch notes simply haven't been updated.

### 3. Who reviews core-domain changes?

**Short answer: Ping Tom Weston first.**

- **team-conventions.md** says "Changes to `core-domain/` need a review from Priya."
- **Tom Weston and Andy K. in Slack** (Mar 2026): Tom took over day-to-day core-domain reviews when Priya moved to a full architecture role. Tom will escalate to Priya for structural decisions.

**Trusted source:** Slack (more recent). For a new domain event within an existing aggregate: ping Tom in `#core-domain-reviews`. For a structural change to the domain model itself: plan a conversation with Priya, not just a PR comment.

### 4. Is the 85% coverage requirement enforced?

**Short answer: Yes, it's a hard CI gate.**

- **team-conventions.md** describes it as a "target."
- **Leila Mohr in Slack** (Apr 2026) confirmed the Platform team enabled JaCoCo threshold enforcement in CI in Q1 2026 (announced in `#eng-general` on March 10). 85% minimum per module, no exceptions.

**Trusted source:** Leila Mohr (more recent, direct knowledge of CI configuration). If your PR fails CI with a coverage message, you need to write more tests — it's not a misconfiguration.

---

## Things That Couldn't Be Confirmed

### Istio service mesh status

arch-notes.md says Istio was "planned for Q1 2026." As of the writing of this guide (May 2026), Q1 has passed. The notes say mTLS is not yet enforced between services. Whether Istio has shipped or is still pending is unknown from the available sources — the documents don't mention it after the planning note.

**Why unclear:** No Slack thread or convention update addresses this after the Q1 deadline. Before writing any code that depends on service-mesh features (mTLS, traffic management), ask in `#eng-general` or check with your team lead.

### Lineage query migration (MER-3421)

arch-notes.md notes that dataset lineage queries degrade quadratically on deep graphs and that ticket MER-3421 tracks moving this to a graph database — but it was not on the H1 roadmap. No Slack thread updates this.

**Why unclear:** The H1 window (through June 2026) is near its end. This ticket may have moved, been deprioritized, or be in flight. Check MER-3421 in Jira before touching lineage-related code.

### collector-svc Datadog dashboard

Leila confirmed in Slack that there's an "Ingestion API - Production" dashboard in Datadog. Tom mentioned a "Collector Service - Production" dashboard, but he also believed collector-svc was still live (it isn't). That dashboard may have been removed or renamed.

**Why unclear:** No source confirms what happened to collector-svc monitoring after decommissioning. Don't rely on the collector-svc dashboard. Use "Ingestion API - Production."

### Flyway destructive migration process

arch-notes.md says destructive database migrations require a review from the Core team and a documented rollback plan. It doesn't specify who "the Core team" is for this purpose, and no Slack thread clarifies it.

**Why unclear:** "Core team" appears to mean engineers with broad platform ownership (Priya, Tom, Leila), but this isn't stated explicitly. Before running any destructive migration, confirm the review process with your team lead.
