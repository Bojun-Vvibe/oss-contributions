---
pr: 24450
repo: anomalyco/opencode
sha: b05be71dfe2fa948f8d74c922e0fb963c182e483
verdict: needs-discussion
date: 2026-04-26
---

# anomalyco/opencode #24450 — feat(agent): add orchestrate subagent for task decomposition

- **Author**: kuro-toji
- **Head SHA**: b05be71dfe2fa948f8d74c922e0fb963c182e483
- **Link**: https://github.com/sst/opencode/pull/24450
- **Size**: +86 / -0 across 3 files. New built-in agent `orchestrate` registered in `agent/agent.ts`, new system prompt `agent/prompt/orchestrate.txt` (66 LOC), 2-line addition to `tool/task.txt` describing when to use it.

## Scope

Adds a new built-in `orchestrate` subagent whose entire job is to call the `Task` tool multiple times in parallel. Permission overlay: `question: allow`, `task: allow`, `todowrite: allow`. Mode `subagent`, `native: true`. Prompt instructs the agent to decompose a task into 2-5 subtasks, dispatch each through `Task` with an appropriate `subagent_type`, and synthesize results.

## Specific findings

- `agent/agent.ts:188-205` — registration block. Permission merge order is `defaults → fromConfig({question: allow, task: allow, todowrite: allow}) → user`. That means a user override can re-tighten task perms, which is correct. Good.
- `agent/agent.ts:198` — `description` is the surface that shows up in agent listings and influences when the primary agent reaches for this subagent. The current text reads "Use this when a task involves multiple independent components, can be parallelized for speed, or is too large for a single agent." — that's overly broad. Most non-trivial tasks meet at least one of those criteria, so the primary will reach for `orchestrate` constantly, which means: every task adds an extra LLM call (the orchestrator) before any work happens. Tighten the description: e.g. "Use ONLY when the user explicitly requests parallel decomposition, or when independent subtasks each require >5 tool calls".
- `agent/prompt/orchestrate.txt:7-11` — "Identify which agents are best suited for each subtask" but the prompt's "Agent Types Reference" table at `:35-41` lists only `explore`, `general`, `build`, `plan`. `general` is not a built-in agent in `agent.ts` (the built-ins are `general`-equivalent under different names). Verify that `general` is actually a valid `subagent_type` value passed to `Task` — if not, the orchestrator will produce dispatch calls that fail at the Task tool with "unknown subagent_type". Quick check needed.
- `agent/prompt/orchestrate.txt:43-49` — example shows 4 parallel subtasks for an OAuth migration. The example explicitly violates the next bullet point: "Handle dependencies: if Task B needs output from Task A, execute sequentially". OAuth migration has obvious sequential dependencies (research → design → implement → wire client). Either fix the example to show a genuinely parallel decomposition (e.g. "split a 200-file rename into 4 directory chunks") or annotate the example with "step 1 must run before steps 2-4".
- `agent/prompt/orchestrate.txt:53-58` — guideline "Trust subagent outputs" is dangerous. If a subagent fails or returns a non-answer, the orchestrator should NOT just fold it into the synthesis. Reword to "Validate subagent outputs against the success criteria you set; re-dispatch if a subagent reports an error".
- `tool/task.txt:5-7` — adds "When you need to break down a complex task into parallel subtasks using the `orchestrate` agent." The `Task` tool prompt now says "use orchestrate to break down" while the `orchestrate` agent's whole purpose is to call `Task`. That's a mutual-recursion footgun: the primary calls Task(orchestrate), orchestrate calls Task(general/explore/etc.). Each Task call is a fresh LLM context. There's no max-depth guard visible in this diff — verify there's a recursion limit elsewhere; if not, a misconfigured prompt could spawn `orchestrate → orchestrate → orchestrate`. The permission overlay grants `task: allow` to orchestrate itself, which makes self-recursion legal.
- `tool/task.txt:25` — in the example block, `"orchestrate"` is listed alongside the FICTIONAL `"code-reviewer"` and `"greeting-responder"`. The header on line 24 explicitly says these are "fictional examples for illustration only - use the actual agents listed above". Mixing a real agent name into the fictional-examples block is misleading; the model could be confused about whether `orchestrate` is real or example.
- No tests. Adding a new built-in agent should at minimum have an existence/registration test that round-trips through agent listing. Easy to add given the existing `agent.ts` test surface.

## Risk

Medium. Functionally additive (zero deletions) but the design has two sharp edges:
1. **Cost / latency multiplier**: orchestrator is itself an LLM turn before any subagent runs. If the description draws the primary agent to it for routine tasks, every interaction becomes ≥2× the LLM calls.
2. **Recursion exposure**: `orchestrate` has `task: allow` and the prompt instructs it to call `Task`. No depth cap is visible in the diff.

## Verdict

**needs-discussion** — the feature direction is reasonable, but (a) the agent description needs to be much narrower so the primary doesn't over-reach for it, (b) the prompt's example currently teaches the wrong lesson (sequential task framed as parallel), (c) confirm a recursion depth guard exists for `Task`-within-`Task`, and (d) verify `general` is a real subagent_type. None of these are blockers individually but together they suggest the rollout plan needs a maintainer call on whether this should land as a built-in or as an opt-in plugin.
