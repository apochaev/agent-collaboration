# #eng-questions — Selected Threads
_Exported for onboarding reference. Covers February–April 2026. Names preserved; messages lightly
edited for length. Threads are ordered by date._

---

## Thread 1 · Feb 26 · Database access

**Jamie Singh** [9:00 AM]
How do I get read access to the production database? I need to check some query patterns for
MER-1601.

**Priya Nair** [9:45 AM]
Post a request in #data-access with: your name, team, what you need access to (full read replica
or specific tenant schemas), and the Jira ticket. Production read access usually takes 1–2 business
days to provision.

**Jamie Singh** [9:47 AM]
Just need read on a few tenant schemas to look at query patterns. Not writing anything.

**Priya Nair** [9:50 AM]
That's fine. Tag @Leila Mohr as approver in the #data-access request. Credentials will come from
Vault once approved.

---

## Thread 2 · Feb 28 · Local dev setup

**Jamie Singh** [2:45 PM]
What's the recommended way to run the stack locally? The README in core-backend mentions both
`make dev` and `docker-compose up` and I'm not sure which one I should be using.

**Andy K.** [2:52 PM]
`make dev` is the one. It wraps docker-compose but handles a few things docker-compose alone
doesn't: injects the dev Vault token so services can pull secrets, sets up local Kafka with the
right topic config, and seeds test customer data so you can actually use the UI end to end.

If you run docker-compose up directly, you'll get containers running but services will fail on
startup because the Vault config isn't there.

**Tom Weston** [3:04 PM]
I've always just done `docker-compose up -d` directly in the service directory and it works fine
for me. You lose the test data seeding but if you're only touching one service it's usually enough.

**Andy K.** [3:09 PM]
Tom's way works when you know what you're doing and you're focused on one service, but for someone
new I'd stick with `make dev`. Less "wait why can't it find the secret" time.

**Jamie Singh** [3:12 PM]
Running `make dev` now, thanks. Taking a while on first run.

**Andy K.** [3:13 PM]
Yeah it pulls a bunch of images the first time. Should be much faster after that. Ping me if you
hit anything weird.

---

## Thread 3 · Mar 5 · RFC process

**Jamie Singh** [10:20 AM]
I'm working on MER-1589, which adds a new optional field to the external API response. Do I need
to file an RFC for this?

**Leila Mohr** [10:35 AM]
Yes. Any change to the external API needs an RFC — even adding an optional field, because some
customers parse responses strictly. Template is on Confluence.

**Jamie Singh** [10:37 AM]
I'm trying to get to Confluence but it says I need SSO access. I think IT is still setting up my
account. Is there another way to find the template?

**Leila Mohr** [10:42 AM]
IT should have auto-created an SSO ticket when you started — if not, file one now via the IT
portal (that one doesn't need SSO). Usually takes 2–3 business days. For now, ask Andy or your
team lead — they'll have the template.

**Andy K.** [10:48 AM]
DM'd you the template. Your IT SSO ticket should be in there already; if not, file one at
it-help.meridian.internal. That site doesn't need SSO.

---

## Thread 4 · Mar 14 · collector-svc status

**Jamie Singh** [9:12 AM]
Quick question — is `collector-svc` still running in production? The architecture notes mention it
alongside `ingestion-api` and I want to make sure I'm looking at the right service for the
ingestion flow I'm debugging.

**Tom Weston** [9:24 AM]
Yeah, collector-svc is still live. We have legacy customers on it — long-term contracts that
predate the ingestion-api migration. They're in maintenance mode but still routing through
collector-svc. Check the Datadog dashboard "Collector Service - Production" if you want to see its
metrics.

**Leila Mohr** [10:03 AM]
@Tom actually that's not right anymore. collector-svc was fully decommissioned in January — the
last migration batch went through during the holiday week. All customers are on ingestion-api now.
The maintenance-mode plan was superseded when we finished early. @Jamie — ingestion-api is the
only entry point. The arch notes are out of date on this.

**Tom Weston** [10:11 AM]
Oh, I thought we still had the Thornfield account on the old path. My mistake. Thanks for the
correction.

**Jamie Singh** [10:14 AM]
Thanks! So ingestion-api logs are what I should be looking at. Is there a dashboard for it?

**Leila Mohr** [10:18 AM]
Yes — "Ingestion API - Production" in Datadog. Also `grafana.meridian.internal/d/ingestion-overview`
for the infra view (VPN required).

---

## Thread 5 · Mar 22 · core-domain ownership

**Jamie Singh** [4:05 PM]
I need to add a new domain event to `core-domain/` as part of MER-1623. The team conventions doc
says to ask Priya for a review, but I see some recent PRs in core-domain reviewed by Tom. Who's
the right person to loop in?

**Andy K.** [4:17 PM]
Priya's still the right call for big architectural decisions in core-domain, but Tom handles the
day-to-day reviews now. For a new domain event I'd ping Tom directly — he'll know if it needs
Priya's eyes too.

**Tom Weston** [4:31 PM]
Yeah, ping me. I took over regular reviews when Priya moved to a full architecture role. She's
still involved for structural changes to the domain model, but for a new domain event within an
existing aggregate, that's me. Tag me as reviewer in the PR and @mention in #core-domain-reviews.

**Jamie Singh** [4:35 PM]
Perfect. Will do.

---

## Thread 6 · Apr 2 · coverage gate

**Jamie Singh** [11:30 AM]
My PR just failed CI with: "Coverage threshold not met (83.4%, minimum 85.0%)". Is the 85%
coverage number actually enforced by CI or is it a configuration mistake? The team conventions doc
describes it as a "target."

**Dan Reyes** [11:44 AM]
It's a target, not a hard gate. I think Leila may have accidentally turned on enforcement at some
point. I wouldn't stress about it — I've had PRs merge with coverage below 85% before. Just add a
note in the PR explaining why coverage dropped.

**Leila Mohr** [11:59 AM]
@Dan it's definitely enforced. The platform team enabled JaCoCo threshold enforcement in CI last
quarter — 85% minimum per module, no exceptions. It was announced in #eng-general on March 10
(search "coverage gate enforcement"). @Jamie you need to write tests to get the build green. Sorry
for the confusion from the doc wording.

**Dan Reyes** [12:03 PM]
Ah, must have missed that announcement. Good to know.

**Jamie Singh** [12:05 PM]
Okay, understood — I need to write tests. Is there a guide for the @IntegrationTest annotation?
I haven't seen it before.

**Andy K.** [12:22 PM]
Look at `processor-svc/src/test/java/io/meridian/processor/integration/` for examples. The
annotation just adds it to the `integrationTest` Gradle source set so it doesn't run with
`./gradlew test`. Testcontainers spins up real Postgres + Kafka for the suite. The setup class is
`BaseIntegrationTest.java`.

---

## Thread 7 · Apr 10 · Kafka topic creation

**Jamie Singh** [2:30 PM]
Does creating a new Kafka topic require the RFC process, or is that separate?

**Leila Mohr** [2:45 PM]
Separate. New topics go through a Design Review, not an RFC — the RFC is for external API
contracts. Open a Terraform PR in the `platform-infra` repo and tag the Platform team. The PR
template has a checklist covering partition count, retention policy, and schema registration.

If the new topic is part of a cross-service API change, you'd file an RFC for the contract and a
Terraform PR for the topic creation — those are parallel tracks, not sequential.

---

## Thread 8 · Apr 18 · scheduler-svc jobs

**Jamie Singh** [11:00 AM]
The architecture notes say scheduler-svc uses Quartz for job scheduling. I'm trying to add a new
scheduled job and the code doesn't look like what the Quartz docs describe. Am I looking at the
right thing?

**Tom Weston** [11:15 AM]
You're in the right place but the arch notes are out of date. We migrated away from Quartz in
March — scheduler-svc now uses Spring `@Scheduled` with Redis distributed locking to prevent
duplicate execution across instances. Look at the existing `@ScheduledJob` beans in `scheduler-svc`
as examples. It's much simpler than Quartz.

**Jamie Singh** [11:18 AM]
That explains it. Is there documentation on the Redis locking implementation?

**Tom Weston** [11:25 AM]
Not formal docs yet, but `RedisJobLock.java` in the scheduler package has a detailed javadoc
comment explaining the approach. Lock key is `scheduler:<job-name>`, TTL is configurable per job
via the `@ScheduledJob` annotation parameters.
