# Self-Improvement Baseline
Date: 2026-05-21

Evaluation of `codex-collaborate` against four criteria before the self-improvement loop runs. Before state and after state are both below.

---

## Before State

### Criterion 1: Manual steps required per session

Counting distinct actions that require explicit execution (not thinking or reading — actual writes, commands, decisions that produce an artifact):

| Step | Action | Required? |
|------|--------|-----------|
| Step 0 | Form and send one clarifying question | Conditional |
| Pre-flight | Run 3 bash verification commands | Always |
| Step 1 | Complete the task (the actual work) | Always |
| Step 2 | Create SESSION.md + write Round 1 | Always |
| Step 3 | Run timeout invocation command | Always |
| Step 3 fallback | Copy Codex section from terminal, append to SESSION.md | If read-only sandbox |
| Step 4 | Locate and read Codex response (Inbox or terminal) | Always |
| Step 5 | Decide what to apply, apply changes | Always |
| Step 6 | Append Final Synthesis to SESSION.md | Always |
| Step 7 | Present result to user | Always |

**Count**: 7 mandatory mechanical steps (happy path), 9-10 with fallback.

**Gap**: Pre-flight is 3 separate commands that could be scripted into one. The fallback path (read-only sandbox) adds 2-3 steps that are fully manual and error-prone — you have to visually locate the right section in terminal output and transcribe it. This is the highest-friction point in the current workflow.

---

### Criterion 2: SESSION.md clarity (readable cold)

Current format:
```
# Session: [task]
Date: YYYY-MM-DD

## Round [n] — [Agent Name]
State: working | handoff | done

### Findings
### Open Questions / Gaps
### Handoff Prompt
---

## Final Synthesis — Claude
State: done
### Result
### What Changed in Round 2
### Residual Open Questions
```

**What works**: Round structure is clear. State values are defined. The append-only rule means later rounds don't overwrite earlier ones. Reading linearly gives you the full history.

**What doesn't work**:
- **No document-level status**: To know if a session is active or complete, you must scroll to the end. If the file is long, "where are we?" is not immediately answerable.
- **No timestamps on rounds**: You can't tell how old a handoff is. A `State: handoff` from three days ago looks identical to one from five minutes ago.
- **Naming inconsistency**: Round sections use `## Round N — Agent Name`, but the final synthesis uses `## Final Synthesis — Claude` — a different naming convention that breaks the scan pattern.
- **No document-level summary**: If someone opens SESSION.md and wants to understand the task and result in 30 seconds, there's no place to look. They have to read the whole file.

---

### Criterion 3: Prompt quality (handoff prompts specific enough?)

The skill specifies the handoff format:

> "Codex — your turn. I found [X]. I didn't look at [Y]. Correct anything wrong. Add [Z]."

And states: "The handoff prompt must be specific. 'Review this' is not a handoff prompt."

**What works**: The format string is good. "I didn't look at Y" is the most important part — it gives Codex explicit targets rather than asking it to guess what's missing.

**What doesn't work**:
- **No explicit uncertainty field**: The skill asks for "open questions / gaps" (things not covered) but not for "uncertain claims" (things covered but possibly wrong). These are different. Codex should be told specifically which claims Claude is uncertain about, not just which areas were left blank.
- **No adversarial framing**: The handoff says "correct anything wrong" but doesn't say "challenge any claim you think is weak, even if you're not certain." Verification mode produces corrections; challenge mode produces independent assessment. Both are useful; only verification is currently prompted.
- **No examples in the skill**: The skill gives the format string but no example of a good vs. bad handoff. A bad handoff ("review this and add what's missing") would satisfy the letter of the format but not the spirit.

---

### Criterion 4: Convergence speed (rounds to reach a good result)

**Evidence from this project**: Every session converged at Round 2. The readability research task (Phase 4) reached a final synthesis at Round 2. All Codex skill reviews converged at Round 2 (State: done). The configured round limit (default: 3) was never tested.

**What works**: The round limit stops runaway sessions. Tasks framed with specific handoff prompts converge quickly.

**What doesn't work**:
- **No convergence signal**: The skill doesn't define what "converged" looks like. A practitioner might not recognize a Round 2 that's already done and continue to Round 3 unnecessarily.
- **No early-stopping guidance**: If Codex's Round 2 response says "confirmed, nothing to add," the skill should tell you to stop there, not write a pro-forma Round 3.
- **No delta tracking**: There's no requirement for Codex (or Claude in Round 3) to state explicitly what's genuinely new vs. what's restating the previous round. Without this, it's easy to miss that a round is just treading water.

---

## Summary: Gaps to Address

| Criterion | Current Score | Key Gap |
|-----------|--------------|---------|
| Manual steps | 7-10 steps | Pre-flight could be one command; read-only fallback is fully manual |
| SESSION.md clarity | Adequate | No document-level status; no timestamps; inconsistent naming |
| Prompt quality | Good | No explicit uncertainty field; no adversarial framing instruction |
| Convergence speed | Fast (Round 2) | No convergence signal; no delta tracking; early-stopping undefined |

---

## After State

Two rounds (Claude + Codex) + Final Synthesis. Eight changes accepted, two deferred.

### Criterion 1: Manual steps required per session

No change to step count — v2 does not reduce mechanical steps. The pre-flight, invocation, fallback, and synthesis steps are identical. What changed: the fallback path is now explicitly documented for macOS (`perl -e 'alarm 120; exec @ARGV'`) and the "Codex pass skipped" path is clearer.

**Gap addressed**: macOS `timeout` incompatibility documented; full fix (pre-flight API check, retry policy) deferred.

---

### Criterion 2: SESSION.md clarity (readable cold)

**Before**: No document-level status, no timestamps, naming inconsistency, no summary.

**After**:
- `### What's New` leads every Round 2+ — the first thing a cold reader sees tells them what changed
- `### Uncertain Claims` (capped 0–3, allow `None`) separates "things I might have wrong" from "things I didn't cover"
- `### Verification Targets` replaces `### Open Questions / Gaps` — more actionable framing
- `### Decisions` in Final Synthesis (Accepted/Rejected/Deferred) replaces `### What Changed in Round 2` — makes the synthesis useful for retrospectives, not just output

**Still deferred**: Document-level `Status: active | done` header (drift risk without automation), mandatory timestamps.

---

### Criterion 3: Prompt quality (handoff prompts specific enough?)

**Before**: No explicit uncertainty field, no adversarial framing, no examples.

**After** (handoff format):
> "Codex — your turn. I found [X]. I'm uncertain about [U]. I didn't look at [Y]. Correct anything wrong. Challenge claims that are weak, underspecified, or unsupported, even if I sounded confident — but preserve the original claim unless you can name the evidence, test result, source, or reasoning that justifies changing it. Add [Z]."

Key corrections from Codex:
- `### Uncertain Claims` gives Codex specific targets (not just gaps)
- "Structured challenge + evidence requirement" replaces generic "adversarial framing" — literature supports evidence-backed dissent for sequential workflows, not generic adversarial debate
- Anti-rubber-stamp and anti-over-revision rules bound both directions

---

### Criterion 4: Convergence speed (rounds to reach a good result)

**Before**: All sessions converged at Round 2 empirically, but no convergence signal in the skill — a practitioner might not recognize early convergence.

**After**:
- `### What's New` is the convergence signal — if thin or `None — converged`, stop
- Convergence as decision rule added to Step 5: stop and synthesize when no material correction, new evidence, or unresolved decision remains
- Anti-rubber-stamp rule prevents false convergence (Codex confirming without stating why)

---

### Summary: Before vs. After

| Criterion | Before | After |
|-----------|--------|-------|
| Manual steps | 7–10 steps | Unchanged — macOS fix documented, full fix deferred |
| SESSION.md clarity | No status, no timestamps, inconsistent naming | `### What's New` first; `### Uncertain Claims`; `### Verification Targets`; `### Decisions` |
| Prompt quality | No uncertainty field, verification-only framing | Uncertainty field (0–3); structured challenge + evidence requirement |
| Convergence speed | No signal; no early-stopping guidance | `### What's New` as delta signal; convergence decision rule in Step 5 |

**Key correction from the loop**: "Adversarial framing" (Claude's initial proposal) was narrowed to "structured challenge + evidence requirement" based on Codex's literature review. The distinction matters: generic adversarial prompting in sequential workflows is not well-supported; evidence-backed dissent is. The v2 handoff format reflects this precisely.
