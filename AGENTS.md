# AGENTS.md

General-purpose orientation for AI agents working in this repository.

---

## What This Repo Is

A demonstration and learning path for multi-agent collaboration protocols built on Claude Code. The primary artifact is the `codex-collaborate` skill — a two-agent protocol where Claude and Codex work in sequence to produce better outputs than either pass alone.

The commit history is a replayable learning path: manual relay → shared session file → automated invocation → round limit → self-improvement loop.

---

## Repo Structure

```
CLAUDE.md                          — Claude orientation
AGENTS.md                          — this file
.claude/skills/codex-collaborate/  — full two-agent collaboration protocol
.claude/skills/codex-consult/      — lightweight single-exchange consultation
examples/                          — worked examples of the protocol in practice
tmp/                               — transient session files (gitignored)
```

---

## Working in This Repo

- Session files under `tmp/` are transient and gitignored — do not commit them
- Canonical outputs live in the repo root or `examples/` after user approval
- The skill files under `.claude/skills/` are the source of truth for protocol details

---

## Structured Collaboration Sessions

When Claude initiates a structured collaboration, you will receive a `SESSION.md` file as your prompt. It contains the task brief, current round state, response format, and a `### Handoff Prompt` section with your specific instructions for that session. Follow those instructions — they are self-contained and override general guidance for the duration of the session.

Do not write to canonical repo files during a session — write only to the session's `drafts/` directory or return inline.
