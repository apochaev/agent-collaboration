# Meridian Systems Engineering First-Week Guide

Welcome to Meridian Systems engineering. This guide turns the architecture notes, team conventions,
and recent Slack Q&A into one practical first-week path. It is written for someone new to the
system, so it explains the moving parts before naming the sharper edges.

## How to Read This Guide

The source documents do not always agree. When they conflict, this guide trusts the most recent
and most specific evidence:

- Recent Slack corrections from named owners are treated as stronger than older architecture notes.
  The architecture notes were last updated on 2025-12-10 and are useful, but several Q1 2026
  changes happened after they were written.
- Team conventions from March 2026 are trusted for daily workflow, except where Slack later
  clarifies a specific detail.
- Architecture notes are still the best source for the high-level service map, data flow, and
  system responsibilities unless a newer source directly contradicts them.

## Day One Checklist

- Get access to GitHub, Jira, Slack, VPN, Datadog, ArgoCD, Vault, and Confluence.
- Join the main engineering channels: `#eng-general`, `#new-hires`, `#code-review`,
  `#platform-rfc`, `#data-access`, `#incidents`, and `#core-domain-reviews`.
- Ask your manager to add you to the Datadog organization.
- Confirm your SSO access for Confluence and internal tools. If Confluence is not ready yet, use
  the IT portal at `it-help.meridian.internal`; it does not require SSO.
- Clone the backend repo or the service repo your team uses.
- Run local setup with `make dev`, not raw `docker-compose up`, for your first setup.
- Open the Datadog dashboard for the service you will work on, plus Grafana if you have VPN.
- Pick a small ticket with your manager or team lead.
- Create a branch using `initials/MER-####-short-description`.
- If you are stuck for 30 minutes, ask in `#new-hires`, `#eng-general`, or the relevant channel.

## What Meridian Builds

Meridian is a data observability platform. Customers send events from their data pipelines to
Meridian. Meridian validates those events against customer-defined schemas and freshness rules,
detects anomalies, and sends alerts through Slack, PagerDuty, email, or webhooks.

In production, the platform handles billions of events per day. Most backend services coordinate
through Apache Kafka, a distributed message broker that lets services publish and consume event
streams without calling each other directly.

The simplified flow is:

```text
customer pipelines
  -> ingestion-api
  -> Kafka topic: events.raw
  -> processor-svc
  -> Kafka topics: events.validated and alerts.triggered
  -> notification-svc
  -> Slack, PagerDuty, email, or webhook
```

The dashboard and external API read from `query-api`. Scheduled freshness and report jobs run in
`scheduler-svc`. Internal support and operations tools use `admin-api`.

## Services You Will Hear About

| Service | What it does | First-week notes |
|---|---|---|
| `ingestion-api` | Current event entry point. Authenticates requests, performs minimal structural validation, and publishes to `events.raw`. | This is now the only ingestion entry point according to the March Slack correction. |
| `collector-svc` | Former legacy ingestion service. | Treat it as decommissioned, not active. Older architecture notes still describe it as running, but Leila corrected that in Slack and Tom confirmed his earlier answer was wrong. |
| `processor-svc` | Core event validation, enrichment, freshness state, and volume anomaly detection. | Most critical service. Changes here need careful tests because latency affects alerts. |
| `query-api` | Read-only REST API for dashboard and external API. | External API changes usually need an RFC. |
| `scheduler-svc` | Runs scheduled freshness checks, windowed anomaly checks, reports, and cleanup. | Uses Spring `@Scheduled` with Redis distributed locking now, not Quartz. |
| `notification-svc` | Consumes alert signals and delivers alerts to configured channels. | Retries transient failures with backoff and records delivery status. |
| `admin-api` | Internal support and ops tooling. | Internal only. Auth uses Okta JWTs according to architecture notes. |
| `frontend` | React 18 / TypeScript customer app. | Separate repo: `meridian/frontend`. |

## Local Development Setup

Use `make dev` for your first setup. It wraps Docker Compose and also does the Meridian-specific
work that raw Docker Compose does not do for new developers.

1. Clone the relevant repo.
2. Install the repo prerequisites listed in that repo's README.
3. Connect to VPN if the README or service needs internal access.
4. From the service or backend repo, run:

   ```bash
   make dev
   ```

5. Wait for the first image pulls. The first run can take a while.
6. Confirm that local Kafka topics were created and test customer data was seeded.
7. Start the service or UI workflow described in the service README.
8. If startup fails on secrets or Vault config, check that you used `make dev` rather than
   `docker-compose up`.

Why `make dev`: Andy clarified in Slack that it injects the dev Vault token, configures local Kafka
topics, and seeds test customer data. Raw `docker-compose up` can work for experienced engineers
working on one service, but it is more likely to fail for a first setup because the Vault and seed
data steps are missing.

## Working With Data

Meridian uses PostgreSQL on CloudSQL, with one primary and one read replica per region according to
the architecture notes. Tenant data is separated with PostgreSQL schemas, which are namespaces
inside a database, not separate databases. Shared reference data lives in the `public` schema.

Important tables named in the architecture notes:

| Table | Purpose |
|---|---|
| `public.customers` | Account records, subscription tier, feature flags |
| `public.schema_registry` | Registered schemas and versions |
| `tenant_<id>.events` | Validated events |
| `tenant_<id>.datasets` | Dataset metadata and current status |
| `tenant_<id>.alert_signals` | Alert history and delivery state |

To request production read access, post in `#data-access` with your name, team, requested scope,
and Jira ticket. For tenant-schema access, tag Leila Mohr as approver. Provisioning usually takes
1-2 business days and credentials come from Vault.

Unconfirmed: the architecture notes say PostgreSQL 13, but no newer source confirms the current
database version. Because the same architecture notes are stale in other Q1 2026 areas, verify the
version before doing work that depends on version-specific PostgreSQL behavior.

## Kafka and Contracts

The main Kafka topics are:

| Topic | Producers | Consumers | Retention |
|---|---|---|---|
| `events.raw` | `ingestion-api` | `processor-svc` | 24 hours |
| `events.validated` | `processor-svc` | Analytics/future consumers | 7 days |
| `alerts.triggered` | `processor-svc`, `scheduler-svc` | `notification-svc` | 72 hours |

All Kafka messages use Avro schemas with Confluent Schema Registry. If you change an Avro schema
in `shared/`, add or update contract tests. New Kafka topics require a Design Review and a
Terraform PR in `platform-infra`; topic creation is not handled only by an RFC.

If the topic also changes a cross-service API contract, run both tracks: RFC for the contract and
Terraform PR or Design Review for the topic.

## Branches, PRs, and Reviews

Use this branch format:

```text
initials/MER-####-short-description
```

You may see older branches using full usernames, but initials are the current convention. A Jira
ticket is required; create one if it does not exist.

Most PRs need two approvals. Docs, low-risk config changes, and dependency bumps that do not
change runtime behavior can usually use one approval. Squash merge is the default.

Open a draft PR early if you want feedback on the approach. Once it is ready, request review and
expect about 24 hours for a first pass. Use `#code-review` for review pings instead of DMs.

Avoid risky Friday deployments. The team treats this as a norm rather than a hard rule, but expect
pushback on Friday afternoon changes that could affect production.

## Tests and Coverage

The 85% line coverage threshold is enforced by CI per module. The team conventions call it a
target, but Leila clarified in April that CI now gates on it with no exceptions.

Common test commands:

```bash
./gradlew test
./gradlew integrationTest
```

Use unit tests for isolated classes. Use integration tests for real database or broker behavior.
Some integration tests use H2, and ones that need actual PostgreSQL behavior use Testcontainers.
Look at `processor-svc/src/test/java/io/meridian/processor/integration/` and
`BaseIntegrationTest.java` for examples.

Contract tests use Spring Cloud Contract and are required for Kafka schema changes.

There are no full end-to-end tests against a real cluster yet. For production-bound work, combine
automated tests with manual staging verification.

## Deployments

Merges to `main` trigger automatic ArgoCD sync to staging. ArgoCD is Meridian's deployment
management tool and is available at `argocd.meridian.internal` with VPN.

Production promotion is manual. Open ArgoCD, find your service, sync it, and get approval from
your team lead or the on-call engineer. Prefer business hours, 9 AM-5 PM PT.

For hotfixes:

1. Merge to `main`.
2. Verify in staging.
3. Sync production in ArgoCD.
4. Get on-call approval.
5. Document the change in `#incidents`.

If a deployment causes user-facing impact, the default response is to roll back first and
investigate after. ArgoCD rollback points the service to the previous image tag and syncs it.

## RFCs and Design Reviews

File an RFC before changing a cross-service API or anything that affects how customers integrate
with Meridian. That includes new endpoints, changed request or response shapes, and new Kafka
message schemas.

The RFC template is in Confluence under Engineering -> RFCs -> Template. If your Confluence SSO is
not ready, ask Andy K. or your team lead for the template; Slack confirms this is the expected
workaround for new hires.

New Kafka topics use Design Review and a Terraform PR in `platform-infra`. If the new topic is
part of a contract change, you may need both an RFC and the topic Design Review.

## Key People to Know

| Person | Role in practice | Go to them for |
|---|---|---|
| Priya Nair | Lead Architect | System architecture, major domain model decisions, database access guidance |
| Leila Mohr | Platform/team lead voice in Slack threads | Coverage gate, production access approval, external API/RFC interpretation, corrections to stale docs |
| Andy K. | Senior engineer and new-hire helper | Local dev setup, RFC templates, process questions, where to find examples |
| Tom Weston | Day-to-day `core-domain` reviewer and scheduler contact | Core-domain PRs, new domain events, scheduler implementation details |
| Dan Reyes | Engineer | Useful context, but verify process claims when they conflict with Platform guidance |
| Your team lead | Your first escalation path | Priorities, production approval, RFC direction, access unblockers |
| On-call engineer | Production operations | Hotfix approval, incidents, production deployment questions |

For `core-domain/`, tag Tom for regular reviews. Priya is still involved for structural domain
model decisions. This updates the team conventions doc, which still says to ask Priya for all
core-domain changes.

## Things That Are Unclear From the Inputs

| Claim | Why it is unclear | What to do with that uncertainty |
|---|---|---|
| Current PostgreSQL version | The architecture notes say PostgreSQL 13, but they are from December 2025 and no Slack thread confirms or updates the version. The same notes are demonstrably stale for collector-svc and scheduler-svc. | Do not rely on PG13-specific assumptions. Verify via Platform, CloudSQL configuration, or infrastructure state before version-dependent DB work. |
| Istio service mesh status | Architecture notes say Istio was planned for Q1 2026. No later source confirms whether it shipped. Because several Q1 2026 architecture-note claims are stale, this plan may also be outdated. | For service-to-service networking or mTLS work, verify current mesh status with Platform or `platform-infra` before coding. |
| `admin-api` auth details | Architecture notes say admin auth uses Okta JWTs, and no other source contradicts it, but it is only mentioned once. | Treat Okta as the likely current model, but confirm in `admin-api` docs or code before changing auth behavior. |

## Contradictions Resolved

| Topic | Conflicting sources | Resolution |
|---|---|---|
| Ingestion entry point | Architecture notes say `collector-svc` still handles legacy customers; March Slack says it was decommissioned in January. | Trust Slack. Leila corrected the status and Tom acknowledged the correction. Use `ingestion-api`. |
| Scheduler framework | Architecture notes say Quartz 2.3.2; April Slack says scheduler migrated to Spring `@Scheduled` with Redis locks in March. | Trust Slack. Tom points to current code patterns and `RedisJobLock.java`. |
| Local dev command | README apparently mentions `make dev` and `docker-compose up`; Slack says both can work. | Use `make dev` as the new-hire path because it handles Vault, Kafka topics, and seed data. |
| Coverage threshold | Team conventions call 85% a target; Slack says CI enforces it. | Treat 85% per module as a hard CI gate. Leila references a March 10 enforcement announcement. |
| Core-domain reviewer | Team conventions say Priya; Slack says Tom handles regular reviews now. | Tag Tom for day-to-day reviews. Bring in Priya for structural domain model changes. |
| Kafka topic process | RFC process covers cross-service/customer contracts; Slack says topic creation has a separate Design Review. | Use Design Review plus Terraform PR for topics. Use RFC too when the topic changes a contract. |

## First-Week Success Pattern

By the end of week one, aim to have:

- Run the local stack with `make dev`.
- Read one service README and traced one request or event through the service.
- Opened a small PR with tests.
- Seen the staging deployment path through ArgoCD.
- Joined the review and support channels.
- Asked at least one question in a public channel when you got stuck.

The strongest norm in these documents is simple: do not disappear into confusion. The team expects
new people to ask questions, and the best answers usually happen in channels where the next new
hire can find them too.
