# Review: sst/opencode#25141 — feat(opencode): configurable timeout for task tool

- PR: https://github.com/sst/opencode/pull/25141
- Head SHA: `2806c94e4885e487c845a88d9529e5289a72fd81`
- Closes: #15080 (parent issue collecting #24669, #23296, #13841, #21250, #22252, #18378, #23891)
- Files: `packages/core/src/flag/flag.ts` (+1/-0), `packages/opencode/src/tool/task.ts` (+53/-16), `packages/opencode/test/tool/task.test.ts` (+76/-0)
- Verdict: **merge-after-nits**

## What it does

Adds an optional per-call `timeout: number` (ms) parameter to the Task tool, plus a global
default override `OPENCODE_EXPERIMENTAL_TASK_DEFAULT_TIMEOUT_MS` (default 10 min). On
expiry the running subagent is cancelled and a `<task_error>` block is returned that
echoes the `task_id` so the caller can choose to resume, widen the timeout, narrow scope,
or abort gracefully.

## Notable changes

- `flag.ts:69` — new `OPENCODE_EXPERIMENTAL_TASK_DEFAULT_TIMEOUT_MS` slot, byte-identical
  shape to the neighbouring `OPENCODE_EXPERIMENTAL_BASH_DEFAULT_TIMEOUT_MS` line so the
  pair is symmetric (bash already had a per-tool timeout knob; the Task tool was the
  outlier).
- `task.ts:9` introduces `const DEFAULT_TIMEOUT = Flag.OPENCODE_EXPERIMENTAL_TASK_DEFAULT_TIMEOUT_MS || 10 * 60 * 1000`
  at module scope, then plumbs the per-call override at `task.ts:31-33` via `Schema.optional(Schema.Number)`
  with the default value interpolated into the description string.
- `task.ts:140-150` — negative-timeout guard returns an `Effect.fail(new Error(...))` before
  any work runs. Correct ordering (validates input before resolving prompt parts).
- `task.ts:151-187` — the load-bearing change: replaces the bare `yield* ops.prompt(...)` with
  `Effect.raceAll([ops.prompt(...).pipe(Effect.map(ok)), Effect.sleep(...).pipe(Effect.map(timeout))])`.
  Discriminated `kind: "ok" | "timeout"` is the right shape — both arms produce
  the same outer type so the subsequent `if (outcome.kind === "timeout") { cancel(); return ... }`
  branch is total.
- `task.ts:158-180` — on timeout, calls the existing `cancel()` closure (already in scope from
  the surrounding fn — it was previously only used on abort) and returns the same `title` /
  `metadata: { sessionId, model }` shape as the success path so the caller's downstream
  formatting code is unchanged. The `<task_error>` block lists three explicit recovery
  strategies (retry with larger timeout / narrow scope / report and stop), which is the right
  prompt-engineering shape for a recoverable error surfaced into a model loop.
- `task.test.ts:387-432` — new live test stubs `prompt` as an `Effect.sleep("2 seconds")`, sets
  `timeout: 50`, then asserts (a) `output` contains `<task_error>`, (b) it contains the literal
  `"did not complete within 50ms"` (pinning the user-visible wording), (c) it does NOT contain
  `<task_result>` (pinning the no-result-on-timeout invariant), and (d) the `cancelled` flag flipped
  (pinning that the cancel side-effect actually fired, not just the error path returned).
- `task.test.ts:434-462` — second test pins the negative-timeout exit shape (`Effect.exit` →
  `_tag === "Failure"`).

## Reasoning

This is a real and well-reported bug class — the linked issues span at least 4 distinct
provider stalls. The fix sits at the right layer (the Task tool, where the subagent loop
is awaited) and has byte-identical structural alignment with the existing bash-tool timeout.
The Effect.raceAll pattern is the idiomatic way to cancel-on-timeout in this codebase
(`bash.ts` uses the same shape) and the `cancel()` invocation on the timeout branch closes the
subagent session rather than orphaning it. Test coverage pins the user-visible wording
and the cancel side-effect, both of which are non-obvious regression vectors.

## Nits (non-blocking)

1. `task.ts:140` — guard rejects only negative values. `timeout: 0` is technically allowed
   and would race against an immediately-resolving `Effect.sleep("0 millis")`, deterministically
   producing a `<task_error>` before `ops.prompt` can settle. Either reject `<= 0` or document
   the zero-means-immediate-timeout semantics in the schema description.
2. `task.ts:160` — the `<task_error>` payload includes `task_id: ${nextSession.id}` for resume,
   but `cancel()` was just called on that same session. If the subagent's resume semantics
   require the session to still be live, the resume hint may be misleading. Worth a comment
   stating that `cancel()` only stops the in-flight prompt, not the session record (assuming
   that is in fact the contract).
3. `task.ts:9` — `DEFAULT_TIMEOUT` resolves the env var at module load time. If the env is set
   after import (e.g. by a plugin that mutates `process.env` before the first Task call), the
   override is lost. Bash tool likely has the same shape so consistency is fine, but a one-line
   comment pinning the eager-resolution intent would help.
4. `task.test.ts:399` — the stub `prompt: (input) => Effect.sleep("2 seconds").pipe(...)` uses a
   2-second sleep with `timeout: 50`. The 40× safety margin is fine but adds 50ms of test
   wall-clock; a 200ms sleep with `timeout: 50` would still be 4× and shave runtime.
5. The `Closes #15080` body lists 7 provider-specific reports that all share the same root
   cause; consider also adding a `<task_error>` integration test where the stub `prompt` returns
   an `Effect.never` to pin "actually-hung subagent gets cancelled" rather than just
   "slow subagent gets cancelled" (functionally equivalent but documents intent).
