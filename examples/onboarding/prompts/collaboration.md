# Two AIs, Two Passes

Ask Claude and Codex to work on the same task together. Claude goes first and does its best. Then Codex reads Claude's work, catches anything wrong or missing, and adds what it found. Claude reads that, decides what to keep, and produces the final version.

You don't need to understand how that handoff works — the skill handles it.

## What you need

- Claude Code open in this folder
- Codex installed (`npx @openai/codex --version` should return a version number)

## What to do

Copy everything inside the box below and paste it into Claude Code:

---

```
collaborate with codex on: Read the three documents in `examples/onboarding/inputs/` — `arch-notes.md`, `team-conventions.md`, and `slack-qa.md` — and turn them into a first-week guide for someone joining the engineering team at Meridian Systems.

The guide should:
- Settle any contradictions between the documents (explain which source you trusted and why)
- Flag anything that couldn't be confirmed — and say *why* it's unclear, not just "ask someone"
- Be written for someone who is brand new, not for someone who already knows the system
- Include: a checklist for day one, steps to set up the development environment, and a table of key people to know

Save the final guide to `examples/onboarding/outputs/onboarding-collaboration.md`.
```

---

## What you'll get

Claude will work through the task, then hand it off to Codex for a second look. You'll see both passes happen. When it's done, two things will exist:

- `examples/onboarding/outputs/onboarding-collaboration.md` — the finished guide
- `examples/onboarding/SESSION.md` — a record of what each AI did and what changed between passes

## What's worth looking at

Open `SESSION.md`. You don't need to read all of it — just look for places where the second pass (Codex) changed something from the first pass (Claude). Those changes show what a second set of eyes actually caught.

Then compare the finished guide against the one-AI version from `single-agent.md`. The differences are usually small and specific — not dramatic. That's what real second opinions look like.
