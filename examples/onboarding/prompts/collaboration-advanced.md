# Two AIs, Two Passes — Advanced

The same task, using the codex-collaborate skill with richer source context and sharper requirements. The advanced prompt gives both passes more to work with — provenance, known staleness, explicit uncertainty framing. That changes what the first pass produces and what the second pass has left to catch.

Paste this into Claude Code from the repo root:

---

```
collaborate with codex on: Synthesize the three input documents in `examples/onboarding/inputs/` into a first-week onboarding guide for new engineers at Meridian Systems.

Source documents:
- `arch-notes.md` — written by the lead architect, December 2025. Accurate when written; some facts may be stale.
- `team-conventions.md` — written by a senior engineer. Useful, but written for someone who already knows the system.
- `slack-qa.md` — a Q&A export from #eng-questions over three months. Answers are sometimes contradictory and sometimes outdated.

Requirements:
- Resolve contradictions where the documents give you enough evidence. Name your evidence.
- Flag claims that can't be verified from the inputs alone. Explain *why* they're uncertain — not just "check with someone."
- Write for a new hire, not the architect who wrote the source documents. Remove or explain insider language.
- Include a first-day checklist, dev setup instructions that actually work, and a Key People table that reflects current reality.

Write the final synthesized guide to `examples/onboarding/outputs/onboarding-collaboration-advanced.md`.
```

---

## What to Look For

The collaboration is most visible in `SESSION.md`. Look for places where the second pass changed something specific — not just rewrote for style, but caught something the first pass missed or got wrong.

**PostgreSQL version uncertainty**: The arch-notes give a version number; Slack threads suggest it may be outdated. The first pass tends to carry this forward silently. The second pass is more likely to make it explicit — naming the staleness inference rather than just noting the discrepancy.

**Istio status**: The arch-notes describe a planned rollout with no confirmation it completed. Watch whether the second pass adds reasoning behind the uncertainty flag, or just echoes the first pass's conclusion.

**New-hire accessibility**: The first pass is often written from the same position as the source documents — accurate but inside-out. The second pass tends to catch places where context a new hire wouldn't have is assumed without explanation.

Compare the output against `prompts/single-agent-advanced.md` to see what the same task produces with one pass instead of two. The differences are usually visible in a few specific places — not everywhere.
