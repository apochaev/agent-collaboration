# Engineering Team Conventions
_Andy K., last updated March 2026. Living doc — if something's wrong or outdated, fix it or ping me._

This is the stuff that's not in any ticket or spec. The stuff you figure out from getting a comment
on your first PR or asking a Slack question that makes the channel go quiet for a second. Writing it
down so you don't have to.

---

## Version Control

### Branch naming

We use `initials/ticket-slug` format:

```
ak/MER-1234-fix-auth-token-refresh
tw/MER-1456-add-dataset-lineage-endpoint
lm/MER-1500-upgrade-gke-node-pool
```

Ticket number is required. If there's no ticket, make one. Stale branches get pruned monthly.

Note: you'll also see branches using the full `username/ticket-slug` format (e.g.,
`andyk/MER-456-scheduler-refactor`). Both show up in the repo history.

### Commits

No enforced format, but most people do conventional commits out of habit. At minimum your commit
message should tell a reviewer what changed and why — not "fix" or "wip". Squash merge is the
default when merging a PR.

---

## Pull Requests

### Approvals

Two approvals required for all PRs. Minor exceptions: doc changes, config tweaks, and dep bumps
that don't touch runtime behavior can go with one — use judgment, and when in doubt get two.

### Draft PRs

Open as draft early if you want approach feedback. Don't request reviews until it's ready to merge.

### Review turnaround

Expect 24 hours for a first pass. If you're on a deadline, say so in the PR description. If you're
blocked waiting, ping in `#code-review` (not DMs — someone else might be free to pick it up).

### Friday deploys

Don't. On-call is lighter on weekends and rollbacks are annoying to coordinate when half the team
is offline. This is more cultural norm than hard rule, but you will absolutely get a message if you
merge something risky on a Friday afternoon.

---

## Testing

### Coverage

We target 85% line coverage per module, measured via JaCoCo. This is tracked in CI. If your PR
drops coverage noticeably, you'll hear about it in review. The CI report shows you which lines are
uncovered — usually not hard to fix.

### Test types

Three levels:

- **Unit tests**: Test individual classes in isolation. Mock everything external. Run with the
  default `./gradlew test`. Fast.
- **Integration tests** (tagged `@IntegrationTest`): Hit a real database or message broker. Most
  use H2 in-memory; ones that need actual PostgreSQL behavior use Testcontainers. Run separately
  with `./gradlew integrationTest` — they're excluded from the default test task because they're
  slow. CI runs them on every PR.
- **Contract tests**: Spring Cloud Contract for Kafka schema contracts. Required for any change to
  a Kafka message schema. If you're touching Avro schemas in `shared/`, you need contract tests.

### What we don't have (yet)

E2E tests against a real cluster. On the roadmap. In the meantime: integration tests + manual
verification in staging before promoting to prod.

---

## Core Domain

Changes to the `core-domain/` module need a review from Priya. She's the keeper of the domain
model and has strong opinions about what belongs in core vs. application layer.

The core-domain module contains: domain entities, value objects, repository interfaces, and the
domain events that cross service boundaries. If you're adding a new entity or changing a domain
event, that's a core-domain change. Adding a use case or a new query endpoint probably isn't.

Plan for a conversation, not just a PR comment — especially for anything that touches how domain
events are structured.

---

## RFC Process

For any cross-service API change, or any change that affects how customers integrate with us (new
endpoints, changed request/response shapes, new Kafka schemas), you need an RFC filed before you
write code.

RFC template and process: Confluence → Engineering → RFCs → Template. File the RFC and @ mention
the relevant team leads. Review SLA is 5 business days.

Don't let this slow you down on smaller changes — if you're not sure whether something needs an
RFC, ping in `#platform-rfc` and someone will tell you in about 15 minutes.

---

## Deployments

### Staging

Merges to `main` trigger an automatic ArgoCD sync to staging. No approval needed. If staging
breaks, the person who merged is responsible for fixing or rolling back before anyone else merges.

### Production

Production promotions are manual. Open ArgoCD, find your service, click Sync. This triggers an
approval gate — you need a thumbs-up from your team lead or the on-call engineer.

Deploy during business hours (9 AM–5 PM PT) when possible. Avoid Fridays.

For hotfixes: merge to main → verify in staging → production sync (on-call can approve). Document
in `#incidents`.

### Rollbacks

ArgoCD rollbacks are fast — point to the previous image tag and sync. For anything causing
user-facing impact, the default is roll back first, investigate after.

---

## Environments

| Environment | Purpose | Who can access |
|---|---|---|
| `dev` | Personal dev work, ephemeral namespaces | All engineers |
| `staging` | Pre-prod validation, runs latest `main` | All engineers |
| `prod` | Production | Team leads + on-call |

For local development, see the README in each service directory. Most services have a `make dev`
target that starts the dependencies you need.

---

## Channels and Communication

| Channel | Use it for |
|---|---|
| `#eng-general` | General engineering discussion, announcements |
| `#code-review` | Review requests (preferred over DMs) |
| `#incidents` | Active incidents, postmortems, hotfix documentation |
| `#platform-rfc` | RFC eligibility checks, RFC discussions |
| `#data-access` | Database access requests |
| `#new-hires` | New hire questions — no question too basic |
| `#core-domain-reviews` | Non-urgent core-domain PR review requests |

---

## Tooling

- **Grafana**: `grafana.meridian.internal` — infra dashboards. VPN required.
- **Datadog**: APM and logs. Ask your manager to add you to the Datadog org on day one.
- **ArgoCD**: `argocd.meridian.internal` — deployments. VPN required.
- **HashiCorp Vault**: `vault.meridian.internal` — secrets. Your dev token comes from `make dev`.
- **Confluence**: Internal docs, RFCs, runbooks. You'll need IT to provision SSO — takes a few
  days if you're new. If you need something urgently before then, ask your team lead.

---

## Asking for Help

No question is too basic. If you've been stuck for 30 minutes, ask.

Code questions: `#eng-general` or DM whoever last touched the area (`git log` is your friend).
Process questions: `#new-hires`.
Architecture questions: `#platform-rfc` or ping Priya or your team lead directly.
