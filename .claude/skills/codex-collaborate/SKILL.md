---
name: codex-collaborate
description: Two-agent collaboration protocol where Claude produces an initial pass, hands off to Codex for independent verification, then synthesizes. Use when the user asks to "collaborate with codex on <task>" or when a second independent pass would materially improve analysis, plans, drafts, or architecture decisions.
allowed-tools: Read Glob Write Edit Bash(mkdir:*) Bash(date:*) Bash(printf:*) Bash(npx @openai/codex:*) Bash(npx -y @openai/codex:*) Bash(gtimeout:*) Bash(perl:*) Bash(wc:*) Bash(grep:*)
---

# Skill: codex-collaborate
Trigger: `collaborate with codex on <task>`

Two agents, isolated session directory. Claude completes the task first, writes a specific handoff, invokes Codex, reads the response, and writes a final synthesis. The session directory under `tmp/codex-collaborate/` is the only coordination layer — each session gets its own directory, enabling true parallel execution across Claude windows.

---

## Session Directory Layout

Every session lives in its own directory derived from today's date and a slug of the task:

```bash
TASK_TEXT="$ARGUMENTS"
SESSION_SLUG=$(printf '%s' "$TASK_TEXT" | tr '[:upper:]' '[:lower:]' | sed 's/[^a-z0-9 ]//g' | tr ' ' '-' | sed 's/-\+/-/g' | cut -c1-50 | sed 's/-$//')
SESSION_DIR="tmp/codex-collaborate/$(date +%Y-%m-%d)-${SESSION_SLUG}"
mkdir -p "$SESSION_DIR/drafts" "$SESSION_DIR/proposed"
```

```
tmp/codex-collaborate/YYYY-MM-DD-{slug}/
├── TASK.md       ← brief, constraints, acceptance criteria, decisions
├── SESSION.md    ← frontmatter + status + round log (append-only)
├── drafts/       ← r1-claude.md, r2-codex.md, r3-claude.md, ...
└── proposed/     ← final artifact for approval before going to canonical_file
```

`tmp/` must be gitignored — sessions are transient. Parallel Claude windows working on different tasks write to different `SESSION_DIR` paths and never touch the same files.

---

## Step 0 — Interview (question budget: 3)

Before writing TASK.md or doing any work, determine what you don't know. You have **3 questions total** across all pre-task exchanges. Spend them on gaps that would materially change your approach — audience, constraints, canonical output path, or existing artifacts to build on.

If all answers are obvious from the request, skip Step 0, document your assumptions in TASK.md, and proceed.

---

## Step 1 — Write TASK.md

Create `$SESSION_DIR/TASK.md` before doing any work. It is written once and not appended:

```markdown
# Task: {description}

## Objective
{goal — one paragraph}

## Constraints
- {list}

## Acceptance Criteria
- [ ] {item}

## Decisions
{answers from interview — or "No interview needed; assumptions below"}

## Artifacts
- Draft (Claude): drafts/r1-claude.md
- Codex pass: drafts/r2-codex.md
- Final: proposed/{deliverable-name}.{ext}
- Canonical: {absolute path in repo where output lands after approval}
```

The `Canonical` path is where the approved output goes after Step 7. If the task is analysis-only (no discrete file deliverable), write `Canonical: none — findings stay in SESSION.md`.

---

## Step 2 — Initialize SESSION.md

Create `$SESSION_DIR/SESSION.md` with this header:

```markdown
---
session: {slug}
date: YYYY-MM-DD
pattern: shared-state
max_rounds: 3        # ← change this one value to adjust the round limit
canonical_file: {same path as Canonical in TASK.md, or "none"}
---

# Session: {task description}

## Status
**Active agent:** Claude
**Round:** 1/{max_rounds}
**State:** working

## Question Budget
3 total, {N} remaining

## Next Action
{one sentence — what Claude is doing right now}

## Dead Ends / Do Not
- {list anything already ruled out}

## Log
```

SESSION.md is **append-only**. Never overwrite earlier content. Each round appends a summary section; full output goes in `drafts/`.

---

## Step 3 — Complete the task (Claude pass)

Do the task: research, draft, structure, analyze — whatever the task requires. Produce a real result, not a plan for a result.

**For tasks with a discrete artifact** (document, code, plan): write the full output to `drafts/r1-claude.md`.

**For analysis/review tasks** (no distinct output file): write findings inline in the Round 1 section of SESSION.md.

Append to SESSION.md log:
```
### YYYY-MM-DD HH:MM — Claude — Round 1 complete. Output: drafts/r1-claude.md
```

---

## Step 4 — Append Round 1 to SESSION.md

```markdown
## Round 1 — Claude
State: handoff

### Summary
[what was produced — key decisions and conclusions, not full content. Full content is in drafts/r1-claude.md]

### Uncertain Claims
[0–3 specific claims you stated that you're not confident about — or "None". These are targets for Codex to verify.]

### Verification Targets
[what you didn't cover, what Codex should check — specific claims, files, tests, or sources]

### Handoff Prompt
Codex — your turn. TASK.md has the brief. I found [X]. I'm uncertain about [U]. I didn't look at [Y]. Correct anything wrong. Challenge claims that are weak or unsupported, but preserve the original claim unless you can name the evidence or reasoning that justifies changing it. Add [Z]. Write your output to `drafts/r2-codex.md` with a `<!-- codex: [brief description] -->` header. If no writable path is available, return inline.

Format your response as:

```
## Round 2 — Codex
State: handoff | done

### What's New
[what this round adds — be specific. If no material delta, write "None — converged".]

### Findings
[corrections, additions, verifications — with evidence or reasoning for each]

### Verification Targets
[what Claude should resolve before closing — only if handing back]

### Handoff Prompt
[only if handing back — specific and actionable]

---
```

---
```

The handoff prompt must be specific. "Review this" is not a handoff prompt.

Update the `## Status` block:
```
**Active agent:** Codex
**Round:** 2/{max_rounds}
**State:** handoff
**Next Action:** Codex to read TASK.md + SESSION.md + drafts/r1-claude.md, then write drafts/r2-codex.md
```

---

## Step 5 — Pre-flight checks

Before invoking Codex, verify:
- Running from repo root (not a subdirectory)
- `$SESSION_DIR/SESSION.md` exists and `State: handoff` is set
- `$SESSION_DIR/drafts/r1-claude.md` exists (for artifact tasks)
- `npx @openai/codex` is available (`which npx` and `npx @openai/codex --version`)
- SESSION.md is under ~50KB (if larger, pass `$SESSION_DIR/drafts/r1-claude.md` + a focused handoff file instead)

---

## Step 6 — Invoke Codex

```bash
gtimeout 120 npx -y @openai/codex exec < "$SESSION_DIR/SESSION.md" || {
  echo "Codex invocation failed or timed out after 120 seconds."
  echo "Options: (1) retry once, (2) proceed to synthesis from Round 1 only."
}
```

**macOS note**: `gtimeout` requires GNU coreutils (`brew install coreutils`). Without it: `perl -e 'alarm 120; exec @ARGV' -- npx -y @openai/codex exec < "$SESSION_DIR/SESSION.md"`

Codex will either:
- **Write to a file** if filesystem permissions allow — look for a file with a `<!-- codex: ... -->` header. Its output should go into `drafts/r2-codex.md`; move it there if Codex writes elsewhere.
- **Return inline** if the sandbox is read-only — copy the `## Round 2 — Codex` section from terminal output and append it to SESSION.md manually.

If Codex has produced no output after ~2 minutes, it is hung. Kill and choose a recovery path — do not wait longer.

Append to SESSION.md log:
```
### YYYY-MM-DD HH:MM — Codex — Round 2 complete. Output: drafts/r2-codex.md
```

**Fallback (Codex timed out)**: Close the session with `State: done` and a synthesis from Round 1 only. A closed single-pass session is better than a blocked session.

---

## Step 7 — Read Codex's response

Read `drafts/r2-codex.md` in full before doing anything. Identify:
- What did Codex correct (with evidence)?
- What did Codex add that you missed?
- What did Codex challenge — and do you agree?

---

## Step 8 — Convergence check

Before proceeding to synthesis or handing back, check Codex's `### What's New` section.

**Stop and synthesize** if:
- `### What's New` contains no material correction, no new evidence, and no unresolved decision another pass can improve
- `### What's New` is marked `None — converged`

If not converged and rounds remain, hand back. Odd rounds go to Claude (`r{N}-claude.md`), even rounds go to Codex (`r{N}-codex.md`). **The final round (`max_rounds`) must always be `State: done`.** Never hand back from the final round.

---

## Step 9 — Write the final synthesis

Write the synthesized output to `proposed/{deliverable-name}.{ext}`.

Append to SESSION.md:

```markdown
## Final Synthesis — Claude
State: done

### Decisions
[Accepted / Rejected / Deferred — one line each with brief reason]

### Residual Open Questions
[what remains unresolved — even if the task is done]

---
```

Update `## Status`:
```
**State:** AWAITING_APPROVAL
**Active agent:** —
**Next Action:** User reviews proposed/{deliverable-name}.{ext}, then approves to copy to canonical_file
```

Append to log:
```
### YYYY-MM-DD HH:MM — Claude — Synthesis written to proposed/{deliverable-name}.{ext}. Awaiting approval.
```

**After approval**: copy `proposed/{deliverable-name}.{ext}` to the `canonical_file` path declared in TASK.md. Then update SESSION.md status to `COMPLETE`.

For analysis-only tasks (no discrete artifact), write the synthesis directly into the Final Synthesis section of SESSION.md and set `State: done` — no `proposed/` step needed.

---

## Step 10 — Present the result

Summarize the final output for the user. Lead with the result, not the process. Point them to `proposed/{file}` for review.

---

## SESSION.md Round 2+ Format

Codex rounds must open with `### What's New`:

```markdown
## Round 2 — Codex
State: handoff | done

### What's New
[what this round adds — be specific. If no material delta, write "None — converged" and Claude should synthesize.]

### Findings
[corrections, additions, verifications — with evidence or reasoning for each change]

### Verification Targets
[what Claude should resolve before closing]

### Handoff Prompt
[...]

---
```

---

## Protocol Rules

**Round limit**: Maximum `max_rounds` rounds (set in SESSION.md frontmatter). Write the best synthesis you have at the final round and stop.

**Anti-rubber-stamp**: Every Codex round must include at least one correction, stronger framing, verification result, or explicit statement that no material issue was found and why.

**Anti-over-revision**: Do not rewrite a prior claim merely because a different framing is possible. Change only when the new framing is more accurate, better supported, or more useful.

**Concurrency safety**: Never read or write another session's directory. Your session is `$SESSION_DIR` — stay inside it.

---

## What Not To Do

- Do not start work before TASK.md is written
- Do not spend more than 3 questions before the first draft
- Do not write a vague handoff ("review this", "please check")
- Do not overwrite earlier rounds in SESSION.md — append only
- Do not pass full draft content to Codex via SESSION.md — use `drafts/r1-claude.md` and reference it in the handoff prompt
- Do not keep going past `max_rounds`
- Do not challenge a claim without naming the evidence or reasoning that justifies the challenge
- Do not copy to `canonical_file` without user approval
