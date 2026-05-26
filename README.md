# agent-collaboration

This started as a small failure mode: Claude could keep the facts straight but flatten the writing; ChatGPT could get closer to the voice and also start inventing details. The useful move was not to ask one model to be better at everything, but to make two models disagree on purpose: one drafts, the other challenges only what it can justify challenging, and the first decides what survives. This repo turns that into a Claude Code workflow with a shared `SESSION.md` file and a three-round limit, because after that the agents mostly negotiate wording.

---

## What's in the Repo

**Skills** (in `.claude/skills/`):

- `codex-collaborate` — full collaboration session, configurable round limit (default: 3), SESSION.md audit trail
- `codex-consult` — bounded single-exchange consultation, no session file

**Example** (`examples/onboarding/`):

Three messy source documents with intentional contradictions and stale facts. `reference/` has the committed outputs from when this was run — collaboration and solo runs for both Claude and Codex, standard and advanced prompt variants. `prompts/` has four variants: basic and advanced versions for both single-agent and collaboration approaches.

**Docs:**

- `docs/manual-relay.md` — what the workflow looked like before the skill existed
- `docs/round-limit.md` — why round limits exist, and why three is the default
- `docs/self-improvement-baseline.md` — before/after evaluation of the protocol against four criteria

---

## Prerequisites

- [Claude Code](https://claude.ai/code) — installed and authenticated
- Codex via `npx @openai/codex` — requires a ChatGPT Plus (or higher) subscription, or an OpenAI API key. On first run the CLI prompts you to authenticate via either method.

```bash
# Verify both are available
claude --version
npx @openai/codex --version
```

**macOS note**: The skill uses `gtimeout` from GNU coreutils as a timeout wrapper — install via `brew install coreutils`. Without it, the fallback is `perl -e 'alarm 120; exec @ARGV'`. The standard `timeout` command from Linux is not available on macOS by default — this was one of the things that broke silently during development.

---

## Use the Skills in Any Project

The skills are self-contained. Copy them into any Claude Code project:

```bash
cp -r .claude/skills/codex-collaborate /path/to/your-project/.claude/skills/
cp -r .claude/skills/codex-consult /path/to/your-project/.claude/skills/
```

Add `tmp/` to your project's `.gitignore` — sessions land there and are transient:

```bash
echo "tmp/" >> /path/to/your-project/.gitignore
```

Then trigger from Claude Code:

```
collaborate with codex on <task description>
```

Or for a lighter pass:

```
run this by codex: <question or draft>
```

Sessions land in `tmp/codex-collaborate/YYYY-MM-DD-<slug>/SESSION.md` in whichever project invokes the skill.

---

## Run the Onboarding Example

`examples/onboarding/prompts/` has four variants of the same synthesis task:

- `single-agent.md` — a direct prompt to Claude or Codex
- `single-agent-advanced.md` — same task with more specific guidance
- `collaboration.md` — the task as a collaboration trigger
- `collaboration-advanced.md` — the collaboration trigger with more specific guidance

Run any combination and compare outputs against `examples/onboarding/reference/`, which has committed results from solo-Claude, solo-Codex, and the two-agent collaboration — standard and advanced variants for each.

To replay: paste the prompt from `examples/onboarding/prompts/collaboration.md` into Claude Code. The guide lands in `examples/onboarding/outputs/` (gitignored). Your session file lands in `tmp/codex-collaborate/` (also gitignored).

The interesting question is not whether the output is correct — it's whether you can see, in your session file under `tmp/`, what each pass added and why.
