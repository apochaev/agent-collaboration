# One AI, One Pass

Ask an AI to read three messy documents and turn them into a clean onboarding guide.

## What you need

- An AI assistant open in this folder (Claude Code, Codex, or similar)

## What to do

Copy everything inside the box below and paste it into your AI assistant:

---

```
Read the three documents in the `examples/onboarding/inputs/` folder:

- `arch-notes.md` — technical notes from the lead architect
- `team-conventions.md` — how the engineering team works day to day
- `slack-qa.md` — real questions new hires asked in Slack, with answers

Turn them into a first-week guide for someone joining the engineering team at Meridian Systems.

The guide should:
- Settle any contradictions between the documents (explain which source you trusted and why)
- Flag anything that couldn't be confirmed — and say *why* it's unclear, not just "ask someone"
- Be written for someone who is brand new, not for someone who already knows the system
- Include: a checklist for day one, steps to set up the development environment, and a table of key people to know

Save the guide to `examples/onboarding/outputs/onboarding-solo-{agent}.md`, replacing `{agent}` with your AI's name (e.g. `claude` or `codex`).
```

---

## What you'll get

A markdown file at `examples/onboarding/outputs/onboarding-solo-{agent}.md` — a structured first-week guide for a fictional engineering team.

## Compare it

Once you have the output, try the two-AI version in `collaboration.md`. Then open both files side by side. The differences are usually subtle but specific — look at how each version handles something it couldn't fully confirm from the source documents.
