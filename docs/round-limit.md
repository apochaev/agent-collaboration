# Round Limit — Why Limits Matter

The collaboration skill uses a configurable round limit (default: 3, set via `max_rounds` in SESSION.md frontmatter). This document explains why the limit exists, why three is a sound default, and what happens without it.

---

## What Happens Without a Limit

Running a two-agent collaboration without a round limit produces a recognizable failure pattern. It doesn't look like a failure at first.

Round 1 produces real findings. Round 2 produces real corrections and additions — things the first pass missed. So far, the protocol is working.

Round 3 is where it starts to blur. The second agent adds something minor, restates a point from Round 1 in slightly different language, or hedges a finding that didn't need hedging. The first agent responds by acknowledging the restatement and adding a minor qualification of its own.

By Round 5 or 6, the session file is long. The synthesis at the top looks authoritative. But reading both together against Round 1 alone, the delta is small. Most of what matters was decided by Round 2.

The pattern has three observable symptoms:

**Restatement.** Each agent begins rephrasing what the other already said. "As Codex noted..." followed by a summary of what Codex said, followed by a new round with "As Claude noted..." Nothing new enters.

**Micro-qualification.** Findings get softer. "X is true" becomes "X is generally true, with caveats," becomes "X may be true in some cases, depending on context." The hedging increases without the underlying claim changing. The session looks more rigorous. It isn't.

**Decision deferral.** The same open questions survive round after round. Neither agent resolves them, rejects them, or marks them out of scope. The question remains visible, but the collaboration has stopped moving it forward. This is the symptom that makes the loop feel productive while preventing convergence.

These symptoms compound. A session in this state is not improving — it's circling. And each additional round makes it harder to extract the actual result from the session file.

---

## Why Three Is the Default

Three rounds is enough to capture the divergence benefit. The structure has a natural shape: Round 1 creates the artifact. Round 2 adds divergence — corrections, additions, reframings. Round 3 decides: what from Round 2 survives, what the combined result says. Round 4 would reopen a decision that should have been made in Round 3.

The limit is not a quality threshold — it's a stopping rule. The default of three does not mean three rounds are always needed. A task can converge in Round 2 and stop. The limit is a ceiling on how long you run before taking the best result you have.

If a task still needs work after the final round, that does not prove the limit was too low. It usually means the remaining question should become a new, narrower session. The round limit stops one collaboration loop; it does not ban follow-up work.

For example: Round 1 identifies that the protocol needs a stopping rule. Round 2 catches the real risk — the agents are not adding evidence, only qualifying each other. By Round 8, the session contains more polished language, but the actionable conclusion is unchanged: stop after synthesis and record unresolved questions.

When to raise the limit: tasks with multiple independent sub-questions that can't be batched into one handoff, or where each pass surfaces genuinely new evidence (rather than restatements of prior rounds). Set `max_rounds` higher in SESSION.md frontmatter. The symptoms in "What Happens Without a Limit" still apply — a higher limit doesn't prevent them, it just gives more room before they set in.

---

## The Constraint in the Skill

The skill enforces the limit via `max_rounds` in SESSION.md frontmatter (default: 3). The convergence check reads:

> The final round (`max_rounds`) must always be `State: done`. Never hand back from the final round. If the task hasn't converged, write the best synthesis you have with residual open questions.

And the Protocol Rules state:

> Maximum `max_rounds` rounds (set in SESSION.md frontmatter). Write the best synthesis you have at the final round and stop. Diminishing returns are the signal to stop, not a failure.

The second sentence is the important one. Stopping at the final round without full convergence is not a protocol failure. It is the protocol working — recognizing that more rounds won't close the remaining gaps, writing the best synthesis available, and making the residual questions visible.

---

## What to Do When the Final Round Arrives Unresolved

The final round is not where the agents try one more time. It is where the current loop closes. The job is to separate what is known from what remains open:

1. State what is resolved.
2. State what remains unresolved.
3. Explain why it remains unresolved: missing evidence, unresolved disagreement, or out-of-scope work.

Then stop. A visible unresolved question is better than another round that makes the session longer without making the answer sharper. The round limit is not about impatience — it is about preventing performative rigor from replacing decision-making.
