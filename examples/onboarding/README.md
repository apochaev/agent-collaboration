# Example: Engineering Onboarding Guide

This example shows two agents synthesizing three messy source documents into a polished first-week
guide for new engineers.

---

## The Scenario

A fictional company (Meridian Systems, B2B SaaS, ~35 engineers) has accumulated three sources of
truth for engineering onboarding — none of them agree with each other.

**`inputs/arch-notes.md`** — Written by the lead architect. Accurate when written (December 2025),
but partially out of date: a deprecated service is still described as running, the database version
may be wrong, a framework migration is not reflected, and a planned infrastructure feature has
an unverified current status.

**`inputs/team-conventions.md`** — Written by a senior engineer. Useful, but written for someone
who already knows the system: the coverage gate is called a "target" when CI enforces it as a hard
gate, branch naming examples show two formats without saying which is current, and the point of
contact for a key code ownership area has changed.

**`inputs/slack-qa.md`** — A real-looking Q&A export from #eng-questions over three months. A new
hire named Jamie Singh asked the right questions; the answers were sometimes contradictory,
sometimes outdated.

## What the Collaboration Produced

**`reference/onboarding-collaboration.md`** — a synthesized first-week guide that:
- Resolves all contradictions where the inputs provide enough evidence
- Flags the two claims that couldn't be verified from the inputs alone (with explanations of *why*
  they're uncertain, not just "check with someone")
- Is written for new hires, not architects — with the insider language removed or explained
- Has a first-day checklist, dev setup that actually works, and a Key People table that reflects
  current reality

## How the Collaboration Worked

The short version:

**Round 1 (Claude)**: Read all three inputs in parallel, identified four cross-document
inconsistencies and four Slack contradictions. Resolved the ones with clear evidence (collector-svc
decommission, Quartz migration, coverage gate enforcement, core-domain ownership), produced a first
draft, and flagged two uncertain claims — PostgreSQL version and Istio status — for Codex to check.

**Round 2 (Codex)**: Found that Claude's PostgreSQL handling was too vague (passing the
uncertainty through silently rather than making it explicit). Confirmed Istio uncertainty using the
same staleness inference Claude used — but added the explanation of *why* the arch-notes are
suspected stale on that point. Caught four places where the draft assumed context a new hire
wouldn't have: the PostgreSQL schema/database distinction, Kafka not being introduced before the
diagram, squash merge behavior, and ArgoCD named without context.

**Final Synthesis (Claude)**: Accepted all Codex changes — each one named the evidence or reasoning
that justified it. Updated the Key People table based on Codex's observation that Andy K. appeared
as the go-to person in four of eight Slack threads.

## What to Look For

The collaboration value is most visible in the differences between rounds:

- **PostgreSQL version claim**: Claude's draft was appropriately vague but didn't explain the
  uncertainty. Codex made the gap explicit — a small change that makes a meaningful difference to
  someone actually doing DB work.
- **Istio uncertainty handling**: Neither agent had direct evidence. Codex improved the framing
  by explaining the staleness inference pattern, so the reader understands *why* it's flagged,
  not just that it is.
- **The four accessibility edits**: Codex read the draft as a new hire would, from a different
  position than the one used to produce it. That's the useful asymmetry in two-agent work.

**Comparing the solo outputs** (`reference/onboarding-solo-codex-advanced.md` vs
`reference/onboarding-solo-claude-advanced.md`): both agents produce strong results from the
advanced prompt, but with a different emphasis. Codex's output is efficient and operationally
clean — concise setup steps, well-suited as a reference for engineers who already know distributed
systems. Claude's output embeds more engineering judgment — it explains traffic patterns, calls out
the critical path, frames permissions in terms of actual workflows, and anticipates the questions a
newcomer won't know to ask. Codex is easier to skim under pressure; Claude builds organizational
fluency faster. Neither difference is visible from the prompt alone.

## Replay It

To run this example with the agent-collaboration skill:

```bash
cd /path/to/agent-collaboration
# trigger:
# "collaborate with codex on: synthesize the three input documents in examples/onboarding/inputs/
#  into a first-week onboarding guide. Resolve contradictions where the inputs have enough evidence.
#  Flag claims that can't be verified from the inputs alone."
```

The `inputs/` files are the starting point. `reference/` has the committed example outputs.
`outputs/` is where your run will write — it's gitignored, so your results stay local. Your session
file lands in `tmp/codex-collaborate/` (also gitignored).

The interesting question is not whether the output is correct — it's whether you can see, in your
session file under `tmp/`, what each pass added and why.
