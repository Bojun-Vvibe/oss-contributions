# PR #24653 — feat(agent): allow agents to ignore instructions

- **Repo**: sst/opencode
- **PR**: #24653
- **Head SHA**: `2a119434`
- **Author**: Hubedge
- **Size**: +47 / -6 across 7 files
- **URL**: https://github.com/sst/opencode/pull/24653
- **Verdict**: **merge-as-is**

## Summary

Adds an opt-in `ignore_instructions: true` knob on agent config that
short-circuits both `Instruction.systemPaths()` and `Instruction.system()`
for that agent, so the agent runs *without* AGENTS.md / CONTEXT.md /
configured `instructions[]` URLs. Use case: a narrowly-scoped subagent
(formatter, classifier, summarizer) that should not be steered by the
project's broader rules.

## Specific changes

- `packages/opencode/src/config/agent.ts:32-34` — schema field
  `ignore_instructions: Schema.optional(Schema.Boolean)` annotated with
  `"Skip loading instruction files for this agent."`, plus the snake-case
  key registered in `KNOWN_KEYS` at `:62` so the agent loader doesn't
  drop it into `options` (the test at `op-24652` style explicitly asserts
  `options not.toHaveProperty("ignore_instructions")` at
  `test/config/config.test.ts:601`).
- `packages/opencode/src/agent/agent.ts:46` — schema mirror as the
  internal camelCase `ignoreInstructions`, copied across in the layer
  effect at `:253` via `item.ignoreInstructions = value.ignore_instructions ?? item.ignoreInstructions`.
- `packages/opencode/src/session/instruction.ts:23` — new local type
  `type AgentInstructions = { ignoreInstructions?: boolean }` plus
  signature change on the `Interface` at `:56-57`: `systemPaths(agent?: AgentInstructions)`,
  `system(agent?: AgentInstructions)`. Both implementations short-circuit
  at the top:
  ```
  if (agent?.ignoreInstructions) return new Set<string>()   // systemPaths
  if (agent?.ignoreInstructions) return []                  // system
  ```
  and `system()` threads `agent` into its own `systemPaths(agent)` call
  at `:166`.
- `packages/opencode/src/session/prompt.ts:1443` — call site updated
  to pass the agent through: `instruction.system(agent).pipe(Effect.orDie)`.
- Tests:
  - `test/agent/agent.test.ts:138,153` — config-load-with-flag round-trip.
  - `test/config/config.test.ts:586,600,604` — schema validation +
    `options` exclusion assertion.
  - `test/session/instruction.test.ts:273-294` — new test
    `"returns empty when agent ignores instructions"` that writes
    a real `AGENTS.md`, calls both `systemPaths({ignoreInstructions: true})`
    and `system({ignoreInstructions: true})`, asserts both return empty.

## Risks

- **Surface width is exactly two functions** (`systemPaths`, `system`).
  Both gates are at the very top of the implementation — no chance of a
  later-added code path leaking instructions when the flag is set
  without the test catching it.
- **Optional argument is correctly threaded**, the existing default
  call sites (no `agent` argument) get `undefined?.ignoreInstructions`
  which is falsy — so the existing behavior is preserved by-construction
  rather than by separate branch.
- **Naming choice** snake_case in user config (`ignore_instructions`)
  vs camelCase in the runtime type (`ignoreInstructions`) is consistent
  with every other field in `AgentSchema` (`top_p` ↔ `topP`,
  `tools` ↔ `tools`, etc.), so no convention drift.
- **No interaction with `instructions[]` URL list**: the early-return
  in `system()` skips both the local-file path *and* the URL fetch path,
  which is what "ignore instructions" should mean. Worth a one-line
  comment in the docstring or PR body confirming intent for the URL
  case, but the test name + behavior are unambiguous.

## What I learned

The pattern of "add an opt-in carve-out that skips a whole subsystem
for one consumer" is at its cheapest when the subsystem has a single
narrow entry point and a single call site. Here `Instruction.system`
is called exactly once from `prompt.ts:1443` for the agent's system
prompt assembly, so a one-arg threading change is sufficient — no need
for a context-propagation refactor or a per-call-site flag. The
schema-field-as-snake-case-with-runtime-camelCase-mirror is the right
shape because it keeps user-facing config syntactically uniform with
the rest of the agent schema while keeping the runtime type readable.
The test at `instruction.test.ts:273-294` writes a real `AGENTS.md` to
a tmpdir and asserts *both* `systemPaths` and `system` return empty —
that two-arm assertion is what catches the future bug where someone
fixes `system()` but forgets `systemPaths()` (or vice versa), which is
exactly the regression shape this kind of feature attracts.
