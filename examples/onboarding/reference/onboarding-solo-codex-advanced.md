# Meridian Systems Engineering First-Week Guide

_Synthesized from `arch-notes.md` (Priya Nair, Dec 2025), `team-conventions.md`
(Andy K., Mar 2026), and `slack-qa.md` (#eng-questions, Feb-Apr 2026)._

Welcome to Meridian. This guide is written for a new engineer who needs to get productive in the
first week, not for someone who already knows the system. When the source documents disagree, this
guide names the evidence used to resolve the disagreement. When the inputs are not enough to prove
something, it says why.

## What Meridian Does

Meridian is a data observability platform. Customers send events from their data pipelines to
Meridian. Meridian validates those events against customer-defined schemas and freshness rules,
detects anomalies, and sends alerts through Slack, PagerDuty, email, or webhooks.

The main production flow is:

```text
Customer pipelines
  -> ingestion-api
  -> Kafka topic: events.raw
  -> processor-svc
  -> Kafka topic: events.validated
  -> alert signals on alerts.triggered
  -> notification-svc
  -> Slack, PagerDuty, email, or webhook
```

`scheduler-svc` also publishes time-based alert signals to `alerts.triggered`. `query-api` powers
the dashboard and external read API. `admin-api` is for internal support and operations tooling.

## First-Day Checklist

- Get access to GitHub, Jira, Slack, VPN, Datadog, ArgoCD, Vault, and Confluence.
- Ask your manager to add you to the Datadog organization on day one.
- File or confirm your SSO ticket at `it-help.meridian.internal`; Slack says this site does not
  require SSO, and Confluence access may take 2-3 business days.
- Join `#eng-general`, `#new-hires`, `#code-review`, `#platform-rfc`, `#data-access`,
  `#incidents`, and `#core-domain-reviews`.
- Clone the backend repo your team uses. If you work on UI, also clone `meridian/frontend`.
- Run the local backend stack with `make dev`.
- Confirm you can find service logs in Datadog and internal infrastructure dashboards in Grafana
  once VPN is available.
- Pick a small Jira ticket with your manager or team lead.
- Create a branch named `initials/MER-####-short-description`.
- If you are stuck for 30 minutes, ask in `#new-hires`, `#eng-general`, or the relevant service
  channel.

## Dev Setup That Works

Use `make dev` for your first local setup.

```bash
git clone git@github.com:meridian/core-backend.git
cd core-backend
make dev
```

If you need the frontend:

```bash
git clone git@github.com:meridian/frontend.git
```

Why `make dev`: Andy K. explained in the Feb 28 Slack thread that it wraps Docker Compose and also
injects the dev Vault token, creates local Kafka topics with the right configuration, and seeds
test customer data. Running raw `docker-compose up` can start containers but leave services unable
to fetch secrets or use the UI end to end.

Tom Weston said raw `docker-compose up -d` can work when you are focused on one service and already
know the system. Treat that as an experienced-engineer shortcut, not the new-hire path.

First run may be slow because it pulls images. If startup fails on missing secrets or Vault config,
check that you ran `make dev` rather than Docker Compose directly.

Unverified from inputs: the exact required versions of Java, Docker Desktop, Gradle, and Node are
not listed in the three source documents. The team conventions point to each service README for
local development, so version-specific setup should be taken from the repo README rather than this
guide.

## System Map

| Service | What it does | What new hires should know |
|---|---|---|
| `ingestion-api` | Stateless Spring Boot REST service that accepts customer events, validates auth, does minimal structural validation, and publishes to `events.raw`. | This is the current and only customer ingestion entry point. |
| `collector-svc` | Former legacy ingestion service. | Treat it as decommissioned. Do not use it to debug current ingestion issues. |
| `processor-svc` | Consumes `events.raw`, validates schemas, enriches events, updates freshness state, detects volume anomalies, and publishes validated events and alert signals. | This is the most critical service in the platform. Latency here delays alerting. |
| `query-api` | Read-only REST API for the customer dashboard and external API. | External API changes usually require an RFC, even optional response fields. |
| `scheduler-svc` | Runs scheduled freshness checks, windowed anomaly checks, reports, and cleanup. | It now uses Spring `@Scheduled` with Redis locking, not Quartz. |
| `notification-svc` | Consumes `alerts.triggered`, deduplicates alerts, and delivers them through customer channels. | Retries transient failures with exponential backoff up to 3 hours. |
| `admin-api` | Internal support and ops API. | Uses Okta JWTs and is not customer-facing. |
| `frontend` | React 18 / TypeScript single-page app. | Separate repo: `meridian/frontend`. It talks to `query-api` and uses `ingestion-api` for onboarding setup. |

## Key People

| Person | Current role in practice | Go to them for | Evidence |
|---|---|---|---|
| Priya Nair | Lead Architect | Major architecture decisions, structural domain-model changes, and production database access guidance. | Author of architecture notes; Mar 22 Slack says Priya remains involved for structural domain changes. |
| Tom Weston | Day-to-day `core-domain` reviewer and scheduler contact | Routine `core-domain/` reviews, new domain events within existing aggregates, scheduler implementation questions. | Mar 22 Slack says Tom took over regular core-domain reviews; Apr 18 Slack points to Tom for scheduler details. |
| Leila Mohr | Platform lead or platform authority in the available threads | Database access approval, RFC interpretation, coverage gate behavior, corrections to stale platform docs. | Feb 26, Mar 5, Mar 14, Apr 2, and Apr 10 Slack threads. |
| Andy K. | Senior engineer and team-conventions maintainer | Local dev setup, process questions, RFC template help before SSO is ready. | Author of team conventions; Feb 28 and Mar 5 Slack threads. |
| On-call engineer | Current production responder | Production deploy approval, hotfix coordination, urgent rollback decisions. | Team conventions production and hotfix sections. |

Unverified from inputs: formal titles for Leila, Tom, Andy, and the on-call rotation are not given.
The table reflects their demonstrated responsibilities in the source documents, not an HR org chart.

## Data and Storage

The architecture notes describe PostgreSQL on CloudSQL with one primary and one read replica per
region. Tenant data is separated by PostgreSQL schema, such as `tenant_<customer_id>`, rather than
by separate database. Shared reference data lives in `public`.

Important tables named in the architecture notes:

| Table | Purpose |
|---|---|
| `public.customers` | Account records, subscription tier, feature flags |
| `public.schema_registry` | Registered schemas and schema versions |
| `tenant_<id>.events` | Validated events |
| `tenant_<id>.datasets` | Dataset metadata and current status |
| `tenant_<id>.alert_signals` | Alert history and delivery state |

To request production read access, post in `#data-access` with your name, team, requested scope,
and Jira ticket. If you need tenant-schema access, tag Leila Mohr as approver. Priya's Feb 26 Slack
answer says provisioning usually takes 1-2 business days and credentials come from Vault.

Unverified from inputs: PostgreSQL 13, exact region topology, PgBouncer modes, and 90-day event
retention appear only in the Dec 2025 architecture notes. They may still be true, but the same
document is stale about ingestion and scheduling, so do not base version-specific or
infrastructure-sensitive work on those details without checking the live infrastructure source.

## Kafka and Contracts

The architecture notes list three Kafka topics:

| Topic | Producers | Consumers | Retention |
|---|---|---|---|
| `events.raw` | `ingestion-api` | `processor-svc` | 24 hours |
| `events.validated` | `processor-svc` | Analytics or future consumers | 7 days |
| `alerts.triggered` | `processor-svc`, `scheduler-svc` | `notification-svc` | 72 hours |

Adjustment from newer evidence: the architecture notes list both `ingestion-api` and
`collector-svc` as producers for `events.raw`. The Mar 14 Slack thread says `collector-svc` was
fully decommissioned in January 2026, so current production ingestion should be through
`ingestion-api`.

All Kafka messages use Avro with Confluent Schema Registry according to the architecture notes.
If you change Avro schemas in `shared/`, add contract tests. New Kafka topics require Design
Review and a Terraform PR in `platform-infra`; Leila clarified on Apr 10 that this is separate
from the RFC process. If the topic is part of a cross-service contract change, do both tracks.

Unverified from inputs: current partition counts and exact retention settings are only stated in
the Dec 2025 architecture notes. They are plausible but not independently confirmed by newer
Slack or convention docs.

## Branches, PRs, and Reviews

Use branch names like:

```text
js/MER-1601-check-query-patterns
ak/MER-1234-fix-auth-token-refresh
```

The current convention is `initials/ticket-slug`. You may see older branches using full usernames.
Every branch needs a Jira ticket; create one if there is no ticket.

Most PRs need two approvals. Docs, low-risk config tweaks, and dependency bumps that do not touch
runtime behavior can usually use one approval. Open draft PRs early for approach feedback, and
request review when the work is ready. Use `#code-review` for review pings rather than DMs.

Avoid risky Friday deploys. The source docs call this a cultural norm, not a hard rule, but you
should expect pushback on Friday afternoon changes with production risk.

## Testing and Coverage

Use:

```bash
./gradlew test
./gradlew integrationTest
```

Test types:

| Test type | Purpose | Notes |
|---|---|---|
| Unit tests | Individual classes in isolation. | Run by `./gradlew test`. Mock external systems. |
| Integration tests | Real database or message broker behavior. | Tagged `@IntegrationTest`; run with `./gradlew integrationTest`. |
| Contract tests | Kafka schema contracts. | Required for Avro schema changes in `shared/`. |

Coverage is a hard CI gate at 85% line coverage per module. The team conventions describe 85% as
a target, and Dan Reyes initially called it non-blocking in Slack. Leila corrected that on Apr 2:
JaCoCo threshold enforcement was enabled in CI last quarter, with no exceptions. If your PR drops
coverage below 85%, add tests.

For integration test examples, Andy pointed Jamie to
`processor-svc/src/test/java/io/meridian/processor/integration/` and `BaseIntegrationTest.java`.

Unverified from inputs: the exact Gradle task wiring and current CI YAML are not included. The
commands and coverage rule are supported by team conventions and the Apr 2 Slack correction, but
task names should be rechecked if a service README differs.

## Deployments

Staging deploys automatically after merge to `main` through ArgoCD. If your merge breaks staging,
you are responsible for fixing or rolling back before more changes pile on.

Production promotion is manual. Open ArgoCD at `argocd.meridian.internal`, find your service, sync
it, and get approval from your team lead or the on-call engineer. Prefer business hours,
9 AM-5 PM PT, and avoid Fridays.

For hotfixes:

1. Merge to `main`.
2. Verify in staging.
3. Sync production in ArgoCD.
4. Get on-call approval.
5. Document the change in `#incidents`.

For user-facing impact, the team convention is to roll back first and investigate after.

## RFCs, Design Reviews, and Schema Changes

File an RFC before code for any cross-service API change or any change that affects how customers
integrate with Meridian. Leila clarified on Mar 5 that even adding an optional field to an external
API response requires an RFC because some customers parse responses strictly.

The RFC template is in Confluence under Engineering -> RFCs -> Template. If your SSO is not ready,
ask Andy K. or your team lead for the template; the Mar 5 Slack thread says this is the expected
workaround. The team conventions say RFC review SLA is 5 business days.

New Kafka topics use Design Review plus a Terraform PR in `platform-infra`, not just an RFC.
If a new topic also changes a customer or cross-service contract, run the RFC and Terraform review
in parallel.

## Channels and Tools

| Channel or tool | Use it for |
|---|---|
| `#eng-general` | General engineering discussion and announcements |
| `#new-hires` | New hire questions |
| `#code-review` | Review requests |
| `#platform-rfc` | RFC eligibility checks and discussions |
| `#data-access` | Database access requests |
| `#incidents` | Active incidents, postmortems, and hotfix documentation |
| `#core-domain-reviews` | Non-urgent `core-domain/` review requests |
| Datadog | APM, logs, and service dashboards |
| Grafana | Infrastructure dashboards at `grafana.meridian.internal`; VPN required |
| ArgoCD | Deployments at `argocd.meridian.internal`; VPN required |
| Vault | Secrets at `vault.meridian.internal`; dev token comes from `make dev` |
| Confluence | RFCs, runbooks, and internal docs; SSO required |

## Resolved Contradictions

### Is `collector-svc` still live?

No. Use `ingestion-api` for current ingestion debugging.

Evidence: the Dec 2025 architecture notes say `collector-svc` was still serving legacy customers
and that roughly 80% of volume had moved to `ingestion-api`. In the Mar 14 Slack thread, Leila
Mohr corrected Tom Weston and said `collector-svc` had been fully decommissioned in January 2026.
Tom then acknowledged his earlier answer was wrong. The newer, explicit correction wins.

### Does `scheduler-svc` still use Quartz?

No. It uses Spring `@Scheduled` with Redis distributed locking.

Evidence: the architecture notes describe Quartz 2.3.2 with a PostgreSQL-backed job store. In the
Apr 18 Slack thread, Tom Weston said the team migrated away from Quartz in March 2026 and pointed
to `@ScheduledJob` beans and `RedisJobLock.java`. Because this is a later implementation-specific
answer from the person helping with scheduler work, it supersedes the architecture notes.

### Who reviews `core-domain/` changes?

Tom Weston handles routine reviews; Priya Nair remains involved for structural domain-model
changes.

Evidence: the team conventions say Priya reviews `core-domain/` changes. In the Mar 22 Slack
thread, Andy K. and Tom Weston clarified that Tom took over regular reviews when Priya moved into
a full architecture role. Tom said he will know when Priya needs to be involved.

### Is 85% coverage only a target?

No. It is enforced by CI per module.

Evidence: the team conventions call 85% a target. In the Apr 2 Slack thread, Dan Reyes initially
said it was not a hard gate, but Leila Mohr corrected him and said JaCoCo threshold enforcement
was enabled in CI last quarter with an 85% minimum and no exceptions. Dan acknowledged the
correction.

## Claims This Guide Cannot Verify From the Inputs Alone

| Claim | Why uncertain |
|---|---|
| Meridian still processes 4-6 billion events per day with 80,000 events/second peaks. | This appears only in Dec 2025 architecture notes. The inputs contain no newer production-volume data. |
| PostgreSQL is still version 13. | Only the stale architecture notes state the version. No Slack thread or convention doc confirms it after Q1 infrastructure changes. |
| The current GKE cluster layout is exactly `prod`, `staging`, and `dev` in `us-central1`. | The architecture notes state it, but infrastructure is explicitly delegated to Terraform in `platform-infra`, which is not one of the provided inputs. |
| Istio is still only planned and mTLS is not enforced. | The architecture notes predicted Q1 2026 work, and the Slack export covers Q1/Q2-adjacent changes but does not mention service mesh status. This is especially likely to have changed. |
| Kafka partition counts and retention are unchanged. | The architecture notes list them, but topic settings are managed through Terraform PRs, and no provided input confirms the current Terraform state. |
| `query-api` endpoint list is complete. | The architecture notes list key endpoints, not all endpoints, and the public OpenAPI specs are named as authoritative but not provided. |
| Frontend deployment details are unchanged. | The architecture notes say the frontend is served from GCS CDN, but the Slack export does not discuss frontend deployment. |

Use this section as a map of where the source packet runs out. For these areas, the right next
source is the specific authoritative artifact named by the docs: service READMEs for local setup,
OpenAPI specs for service contracts, and Terraform in `platform-infra` for infrastructure state.
