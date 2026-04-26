# sst/opencode PR #24450 — feat(agent): add orchestrate subagent for task decomposition

- **PR:** https://github.com/sst/opencode/pull/24450
- **Head SHA:** `b05be71dfe2fa948f8d74c922e0fb963c182e483`
- **Files:** 3 (+86 / -0)
- **Verdict:** `needs-discussion`

## What it does

Registers a new built-in `orchestrate` subagent in `packages/opencode/src/agent/agent.ts:189-205`
with a 66-line prompt at `packages/opencode/src/agent/prompt/orchestrate.txt`.
The agent is granted `question:allow`, `task:allow`, `todowrite:allow` via
`Permission.fromConfig({...})`, defaults `mode: "subagent"`, `native: true`,
empty `options: {}`. The Task tool blurb at `packages/opencode/src/tool/task.txt:7,27`
gets two new lines pointing toward `orchestrate` for parallelizable work.

## Specific reads

- `agent.ts:189-205` — registration block follows the `general`/`build`/`plan` template exactly; permission merge order `(defaults, fromConfig({...}), user)` is identical to siblings, so user `agent.json` overrides still work as expected.
- `orchestrate.txt:42-45` — Agent Types Reference table lists `explore`, `general`, `build`, `plan`. This is hardcoded into the prompt — if a user adds a custom subagent or ships an `arena`/`compaction` agent, the orchestrator will pick from a stale list.
- `orchestrate.txt:54-58` — example decomposition for "Migrate the authentication system to OAuth2" routes design to `plan` then implementation to two `general` calls. There's no guidance on how the orchestrator decides parallel-vs-sequential beyond a one-line "if Task B needs output from Task A, execute sequentially" — leaving the actual scheduler logic implicit.
- `task.txt:27` — the new example agent description is inserted *above* `code-reviewer` and `greeting-responder` in the `<example_agent_descriptions>` block. That block is explicitly labeled "fictional examples for illustration only" — putting a real registered agent in the fictional-examples block is confusing.

## Why needs-discussion

1. **Task-tool semantics overlap**: `task.txt` already documents how the calling agent should invoke `Task(...)` directly. An "orchestrator" that itself fans out `Task(...)` calls adds a layer of indirection where a primary agent issues `Task(orchestrate)` which then issues N `Task(general)` calls. That's three context windows deep. What's the maintainer-intended boundary between "primary agent should just call Task() N times" vs "primary agent should delegate the planning to orchestrate"? The PR doesn't answer this and the prompt copy doesn't either.
2. **Token cost**: each fanout multiplies token spend; `orchestrate` itself is a full LLM call to produce a 2-5 item plan. For a 2-subtask job this is strictly more expensive than the primary agent calling Task twice directly. Where's the break-even, and should `orchestrate` be gated to N≥3 subtasks?
3. **Hardcoded agent table goes stale**: as noted, the prompt enumerates agent types. Better to inject the live agent list into the prompt at runtime (the way other meta-agents see the available subagents) so user-added agents are reachable.
4. **`question:allow` for a subagent**: the orchestrate prompt never tells the agent to ask the user questions — it tells it to decompose and dispatch. Granting `question` to a non-interactive subagent invites mid-orchestration prompts that break the parent flow. Consider `question: "deny"` unless there's a deliberate reason.
5. **No tests / no eval**: an orchestration meta-agent is exactly the kind of feature that needs a deterministic eval (input task → expected number+shape of subtasks) to prevent silent prompt regressions. None added.
6. **Drive-by `task.txt` example placement**: as noted, listing a real agent in the "fictional examples" block is wrong. Move it to a real Available Agents section.
