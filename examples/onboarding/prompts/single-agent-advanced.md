# One AI, One Pass — Advanced

Ask an AI to read three messy documents and produce a rigorously sourced onboarding guide: resolve contradictions, flag uncertainties with reasoning, and write for a reader who knows nothing.

## What you need

- An AI assistant open in this folder (Claude Code, Codex, or similar)

## What to do

Copy everything inside the box below and paste it into your AI assistant:

---

```
Read the three input documents in `examples/onboarding/inputs/`:

- `arch-notes.md` — written by the lead architect, December 2025. Accurate when written; some facts may be stale.
- `team-conventions.md` — written by a senior engineer. Useful, but written for someone who already knows the system.
- `slack-qa.md` — a Q&A export from #eng-questions over three months. Answers are sometimes contradictory and sometimes outdated.

Synthesize them into a first-week onboarding guide for new engineers at Meridian Systems.

Requirements:
- Resolve contradictions where the documents give you enough evidence. Name your evidence.
- Flag claims that can't be verified from the inputs alone. Explain *why* they're uncertain — not just "check with someone."
- Write for a new hire, not the architect who wrote the source documents. Remove or explain insider language.
- Include a first-day checklist, dev setup instructions that actually work, and a Key People table that reflects current reality.

Save the guide to `examples/onboarding/outputs/onboarding-solo-{agent}-advanced.md`, replacing `{agent}` with your AI's name (e.g. `claude` or `codex`).
```

---

## What you'll get

A markdown file at `examples/onboarding/outputs/onboarding-solo-{agent}-advanced.md` — a first-week guide that shows its reasoning, not just its conclusions.

## What to look for

Compare the result against `reference/onboarding-collaboration-advanced.md`. The differences are usually visible in a few specific places:

**PostgreSQL version uncertainty**: The arch-notes give a version number; Slack threads suggest it may be outdated. A single-agent pass tends to carry the uncertainty forward silently. The collaboration output makes it explicit and explains *why* it's uncertain.

**Istio status**: The arch-notes describe a planned rollout; there's no confirmation it happened. Single-agent output typically notes this as uncertain. The collaboration pass added the staleness inference pattern — explaining the reasoning behind the flag, not just the flag itself.

**New-hire accessibility**: The single-agent pass is usually written from the same position as the source documents — accurate but inside-out. The collaboration pass read it as a new hire would, and caught places where context a new hire wouldn't have was assumed without explanation.

The `SESSION.md` in the collaboration output shows exactly where the two passes diverged and why.
