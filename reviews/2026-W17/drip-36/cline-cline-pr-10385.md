# cline/cline #10385 — fix(claude-code): handle CLI v2.1+ result chunks and error_max_turns

- **Repo**: cline/cline
- **PR**: [#10385](https://github.com/cline/cline/pull/10385)
- **Head SHA**: `86ad259d22679b472708a2f5d74b03661b77686b`
- **Author**: DanMarshall909
- **State**: OPEN (+771 / -131; majority is `package-lock.json` + `tsx` dev-dep churn)
- **Verdict**: `merge-after-nits`

## Context

The Claude Code CLI v2.1 release tightened the `result` chunk shape
in two ways: (a) on success it omits the `result` string property and
emits only the metadata (`total_cost_usd`, `usage`, etc.), and (b) on
turn-cap exhaustion it sends `is_error: true` with
`subtype: "error_max_turns"` and *no* `result` payload. Cline's
adapter pre-fix did
`if (chunk.type === "result" && "result" in chunk)` and so on
v2.1+ the chunk fell through to a `Logger.warn("Unrecognized Claude
Code chunk")` and the turn never finalized usage/cost.

## Design

Core change in `src/core/api/providers/claude-code.ts:184-192`:

```ts
if (chunk.type === "result") {
    if (chunk.is_error && chunk.subtype !== "error_max_turns") {
        throw new Error(`Claude Code returned an error: ${chunk.result ?? JSON.stringify(chunk)}`)
    }
    usage.totalCost = isPaidUsage ? chunk.total_cost_usd : 0
    ...
```

Three correct decisions:

1. **Drop the `"result" in chunk` guard** — that was the bug. The
   `type === "result"` discriminator is sufficient.
2. **`chunk.subtype !== "error_max_turns"`** carve-out — the
   max-turns case is *not* a hard error in cline's semantics; it's a
   normal end-of-turn that the upstream loop already handles. Re-
   throwing it would have surfaced as a noisy user-visible failure
   for what is effectively "I hit my budget, try again".
3. **`chunk.result ?? JSON.stringify(chunk)`** for the genuine error
   path — preserves diagnostic value when v2.1+ omits `result` but
   still emits an `is_error` chunk. Better than the prior
   `${chunk.result}` which would have stringified `undefined`.

The new test injection seam at `claude-code.ts:13-14, 33-34`:

```ts
/** For testing only: inject a custom runClaudeCode implementation */
_runClaudeCode?: (...args: Parameters<typeof runClaudeCode>) => ReturnType<typeof runClaudeCode>
...
const runFn = this.options._runClaudeCode ?? runClaudeCode
```

is the standard "underscore prefix means test-only injection point"
pattern. The `Parameters<typeof runClaudeCode>` /
`ReturnType<typeof runClaudeCode>` typing keeps it honest — if the
real signature changes, the test injection point breaks at compile
time. Acceptable, though I'd prefer a constructor-injected factory
over an option-bag escape hatch (the option-bag form leaks the test
seam into the public type).

## Risks

- **Public-API leakage**: `_runClaudeCode` is in
  `ClaudeCodeHandlerOptions` which is part of the handler's
  contract. Even with the `_` prefix and the doc comment, anyone
  consuming the type sees it. A `Symbol`-keyed seam or a
  package-internal `__test__` export would keep this hidden.
- **Test surface added under `src/integrations/claude-code/run.test.ts`**
  via `proxyquire` (line 891+). The `.mocharc.json` change at line
  10-17 adds `src/integrations/claude-code/run.test.ts` to the spec
  glob and adds `tsx/cjs` and `tsconfig-paths/register` to the
  require list. The new `tsx@4.21.0` dev dep brings in a substantial
  esbuild platform-binary tree (line 70+) — that's the cost of
  running TS test files through a faster loader. Reasonable, but
  worth confirming the `devOptional`/`optional` flags on the
  per-platform esbuild binaries don't bloat installs for everyone.
- **`error_max_turns` semantic** — confirm with the upstream loop
  (`src/core/task/Task.ts` or wherever the turn loop lives) that
  silently returning from this branch *is* what the loop expects.
  If the loop is waiting for either a finalized result or a thrown
  error, falling out of the if-block without setting any
  termination signal could hang the turn. Worth a one-line comment
  pointing at the consumer.

## Verdict

`merge-after-nits` — fix is correct, the test infrastructure is
worth the diff weight, but the public-type leakage of
`_runClaudeCode` and the loop-contract clarification on
`error_max_turns` should land before merge.
