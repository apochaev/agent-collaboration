# Manual Relay — The Before State

Before the shared session file existed, running two agents on the same task required manual relay: copying text from one interface to another, keeping state in your head, and hoping nothing got lost in transit.

This document describes that workflow and shows a concrete example. It is the "before" that makes the improvement visible.

---

## How It Worked

### Step 1 — Prompt Claude, request a handoff prompt

You asked Claude to work on the task and ended the prompt with a specific instruction:

> "At the end, generate a specific prompt I can give to Codex for a second pass."

This was necessary because Claude doesn't know about Codex's interface or expectations. Without the explicit instruction, you'd get a result but no packaged handoff — and you'd have to write the Codex prompt yourself.

### Step 2 — Copy Claude's output

Claude produced its response plus a generated Codex prompt. You copied the Codex prompt manually — usually just the prompt block, not Claude's full output.

### Step 3 — Open Codex, paste the prompt

Codex runs as a VS Code extension or via `npx @openai/codex`. You opened a new session, pasted the prompt, and ran it.

Codex doesn't have Claude's context. Its only input is what you pasted.

### Step 4 — Read Codex's response, take notes

Codex returned its pass. You read it against Claude's output and identified what to keep: corrections, additions, reframings. No file tracked this — it was in your head or a scratch note.

### Step 5 — Manually incorporate or continue

You either:
- Applied Codex's input directly to the task output
- Went back to Claude with "Codex said X — does that change your answer?"

No session state. No audit trail. If you came back the next day, the context was gone.

---

## What Made It Clunky

**Prompt authoring overhead.** Every task required writing a Codex handoff prompt from scratch. The quality of the second pass depended entirely on how well you phrased it. Vague prompts got vague responses.

**Context loss between agents.** Codex only got what you pasted. It couldn't see Claude's reasoning, the source material, or the constraints that shaped the first pass. You were the lossy relay.

**No audit trail.** There was no record of what each agent produced, what was accepted, or why. A week later, you couldn't reconstruct what the collaboration had decided.

**Manual state management.** Knowing where you were in the process — whose turn it was, what had been resolved, what was still open — was your job. If you stopped mid-task, you lost the thread.

**No acceptance boundary.** Codex's suggestions were a list. Some you applied, some you skipped, some you half-applied. Nothing tracked which was which. The final output had no record of what came from Codex vs. what Claude produced originally — and no record of what was rejected and why.

---

## Concrete Example

The exchange below is illustrative — it represents how these sessions typically went, not a verbatim transcript.

**Task**: "What makes writing easy to read?"

**Claude prompt:**
```
Research what makes writing easy to read. Look at readability formulas (Flesch, 
Gunning Fog), writing advice (clarity, sentence length, active voice), and any 
comprehension research you know. Produce a practical answer. At the end, generate 
a specific prompt I can give to Codex for a second pass.
```

**Claude's output (condensed):**

Readability formulas measure surface features: sentence length, syllable count. Flesch-Kincaid gives a grade level. Gunning Fog similarly. These are fast proxies for reading ease.

Writing advice tends to focus on clarity: short sentences, active voice, concrete nouns, cut adverbs. Strunk & White. Plain Language movement. These rules correlate with lower Flesch-Kincaid scores.

What the formulas miss: whether the reader already knows the concept. A technically simple sentence can still be hard to read if the reader doesn't have the background knowledge to parse it. Cognitive load research (Sweller, Mayer) connects this to working memory — familiar concepts take up less of it.

Practical answer: readable writing is writing that requires less working memory to process. That is a function of both the prose and the reader's existing knowledge. The formulas measure half of it.

**Claude's generated Codex prompt:**
```
Codex — your turn. I looked at readability formulas (Flesch, Gunning Fog) and 
common writing advice (short sentences, active voice). I found that formulas measure 
surface features but miss the reader's existing schema. I didn't look at empirical 
comprehension research, eye-tracking studies, or whether there's evidence that 
simpler prose actually outperforms well-structured complex prose for expert readers. 
Correct anything wrong in my summary. Add whatever I missed.
```

**Codex's response (condensed, pasted manually):**

The schema point is real. Sweller's work on cognitive load connects it to working memory: when readers lack the background to chunk a concept automatically, they spend working memory on parsing basics instead of comprehension. Surface simplification helps, but the reader's prior knowledge matters too — it's not just about sentence length.

Eye-tracking research (Rayner et al.) shows that reading ease correlates with fixation duration and regression frequency. Long words and unusual syntax cause more regressions. This gives empirical ground to the "simpler is easier" claim — but it is a processing claim, not a comprehension claim.

Missing from your summary: the sentence-length advice is partially correct but the variance matters more than the mean. Monotonous short sentences are harder to follow than varied-length sentences with clear logical connectors.

**What happened next:**

Took notes on the schema point (good addition), the fixation research (new to me), and the variance point. Went back to Claude with: "Codex said sentence-length variance matters more than the mean — does that hold up with what you found?" Started a new Claude session, re-pasted context, continued.

Total overhead: ~10 minutes of manual relay work on top of the actual task. No record of which version was current.

---

## Why This Led to the Skill

The manual relay worked — the two-pass approach produced better output than either agent alone. But the overhead was high enough that it only happened for high-stakes tasks. A shared session file and a structured handoff format made it cheap enough to run on anything.

Commit 3 introduces the skill that replaced this.
