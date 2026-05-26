# CLAUDE.md

Two agents, working in sequence on the same task. Claude produces the initial pass, hands off to Codex for independent verification and challenge, then synthesizes. Codex proposes; Claude decides what to keep. The benefit is divergence — two passes with different defaults and failure modes, not one pass done twice.

The commit history is a replayable learning path: manual relay → shared session file → automated invocation → round limit → self-improvement loop.

---

## How to Run

Use the `codex-collaborate` skill — it handles session setup, Codex invocation, convergence checks, and the approval flow:

> `collaborate with codex on <task>`

For a single bounded second opinion (no session file, one round only):

> `run this by codex: <question or draft>`

Don't implement the protocol manually — invoke the skill.

---

## Repo Layout

- `CLAUDE.md` — this file
- `AGENTS.md` — Codex orientation
- `.claude/skills/codex-collaborate/` — full protocol: session layout, round format, invocation, convergence
- `.claude/skills/codex-consult/` — lightweight single-exchange consultation
- `examples/` — worked examples
- `tmp/codex-collaborate/` — session files (gitignored, transient)
