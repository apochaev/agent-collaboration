---
name: codex-consult
description: Single-exchange Codex consultation. Use when Claude wants one bounded second opinion on a plan, framing, draft, or small factual sanity check before deciding. Trigger phrase: "run this by codex: <question or draft>".
allowed-tools: Read Write Bash(mkdir:*) Bash(date:*) Bash(npx @openai/codex:*) Bash(npx -y @openai/codex:*) Bash(gtimeout:*) Bash(perl:*)
---

# Skill: codex-consult
Trigger: `run this by codex: <question or draft>`

A single-exchange consultation. One structured handoff, one Codex response, one synthesis. No session file, no round tracking.

---

## When to Use This Skill

Use `codex-consult` when Claude already owns the work and wants one bounded second opinion before deciding: planning reviews, framing checks, draft pressure-tests, small factual sanity checks, or "what am I missing?" questions.

Use `codex-collaborate` when Codex is expected to materially co-produce the output, when research or revision will need multiple rounds, or when the shared `SESSION.md` audit trail matters.

If Codex's response creates follow-up work that needs another exchange, stop the consult. Either make the decision yourself or switch to `codex-collaborate`.

---

## Workflow

### Step 1 — Write the handoff file

Pick a timestamp, create the file, fill in three sections:

```bash
TIMESTAMP=$(date +%s)
mkdir -p "tmp/codex-consult"
HANDOFF="tmp/codex-consult/consult-${TIMESTAMP}.md"
# write the handoff to "$HANDOFF", then invoke in Step 2
```

Three sections:

```
## Context
[What we're working on — 2-3 sentences. Enough for Codex to understand scope without reading the full project. Include constraints, audience, or success criteria here if relevant.]

## What I'm evaluating
[The exact question, decision, draft, plan, or framing Codex should inspect. Include the full text or enough excerpted material for Codex to evaluate without guessing.]

## Specific ask
[What you want Codex to confirm, challenge, or add. Be precise:
- "Confirm whether this framing holds up"
- "Catch what I'm missing in this plan"
- "Challenge any assumptions that don't hold"
Not: "review this" or "what do you think?"]
```

### Step 2 — Invoke Codex

```bash
gtimeout 120 npx -y @openai/codex exec < "$HANDOFF" || {
  echo "Codex invocation failed or timed out after 120 seconds."
  echo "Options: (1) retry once, (2) make the decision yourself and note what Codex would have checked."
}
```

**macOS note**: `gtimeout` requires GNU coreutils (`brew install coreutils`). Without it: `perl -e 'alarm 120; exec @ARGV' -- npx -y @openai/codex exec < "$HANDOFF"`

If Codex has produced no output after ~2 minutes, it is hung. Kill the process — do not wait longer. Retry once if network access looks healthy; otherwise proceed without the Codex pass and note what it would have checked.

Codex will either:
- **Write to a file** if filesystem permissions allow it — look for a file with a `<!-- codex: ... -->` header in a writable location (check your configured Inbox or the project's `tmp/` directory)
- **Return inline** if the sandbox is read-only — response comes back in terminal output

### Step 3 — Read the full response

Read before doing anything. Identify:
- What did Codex confirm?
- What did Codex challenge or push back on?
- What did Codex add that wasn't in the original?

### Step 4 — Synthesize back

Report to the user with this shape:
- **Bottom line:** what you will do differently, if anything
- **Confirmed:** what Codex agreed with, and why it matters
- **Challenged:** what Codex pushed back on, and whether you accept the challenge
- **Added:** what Codex contributed that was not in the original

Lead with the bottom line. Keep it short — this is a consultation, not a document.

---

## What Not To Do

- Do not start a SESSION.md — this is not a codex-collaborate session
- Do not run more than one round — if the response opens new questions, stop the consult or switch to `codex-collaborate`
- Do not write a vague ask ("what do you think?") — it produces a vague response
- Do not summarize Codex's response without making a call: agree, disagree, or note the tension

---

## Worked Example

During planning for this repo, Claude used this pattern before writing the plan: "Are these the right clarifying questions? What am I missing?"

Codex sharpened the questions, caught that the self-improvement loop needed engineering success criteria (not just a narrative beat), pushed back on an over-simple "Claude writes code, Codex writes prose" framing, and added missing questions. The plan changed materially after one exchange.

That is the intended use: bounded consultation that improves Claude's next decision without starting a shared session.
