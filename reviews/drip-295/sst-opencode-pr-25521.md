# sst/opencode PR #25521 — refactor(cli): convert mcp commands to effectCmd

- **Repo:** sst/opencode
- **PR:** #25521
- **Head SHA:** `cc4e88ba64c30d13872f1c61bc291a2b7fbd2384`
- **Author:** kitlangton
- **Title:** refactor(cli): convert mcp list, auth, auth list, logout to effectCmd
- **Diff size:** +245 / -256 across 1 file
- **Drip:** drip-295

## Files changed

- `packages/opencode/src/cli/cmd/mcp.ts` (+245/-256) — collapses the four `cmd({ async handler() { await WithInstance.provide(...) } })` shells around `mcp list`, `mcp auth`, `mcp auth list`, and `mcp logout` into the new `effectCmd` wrapper, so handlers become `Effect.fn(...)` generators that yield against `Config.Service` / `MCP.Service` directly.

## Specific observations

- `mcp.ts:67-93` — `listState()` and `authState()` lose their `AppRuntime.runPromise` wrappers and now return raw `Effect`s. That is exactly the right move now that the handlers are themselves Effects, but it's also a load-bearing API change for any in-tree caller that imported these helpers. Worth a quick `rg "listState|authState"` to confirm nothing outside this file consumed the previous Promise-returning versions.
- `mcp.ts:402-410` — the `Effect.catchCause` + `Cause.squash` block replaces the prior `try/catch` around the OAuth poll. `Cause.squash` returns `unknown`, and the code then narrows with `error instanceof Error`. That's correct, but it silently swallows interrupt and parallel-failure structure that the old `catch (error)` never had to worry about. For an interactive CLI flow this is fine; for posterity, a one-line comment that "interrupts surface as squashed errors and are reported as failures here" would help the next reader.
- `mcp.ts:~410` — `Effect.ensuring(Effect.sync(() => unsubscribe()))` is the right replacement for the old `finally { unsubscribe() }`. Good — `ensuring` runs on success, failure, and interrupt, which is strictly more correct than the previous `finally` (which would not have fired on uncaught sync throws inside the try). This is a real behavioral improvement worth calling out in the PR description.
- The diff is mechanical and net-negative on lines, which is the ideal shape for an Effect migration. No new tests, but there are no behavioral changes the existing test surface wouldn't already cover — the only behavioral delta is the `ensuring`-vs-`finally` upgrade noted above.
- Net-zero risk for the `list`/`auth list`/`logout` paths since they were pure read-then-render; the only path with real semantics is the OAuth poll in `mcp auth`, and the cause handling there reads correctly.

## Verdict: `merge-as-is`

Clean mechanical refactor with one incidental correctness improvement (`ensuring` over `finally`). No tests required. Optional: a sentence in the commit body about the interrupt-handling upgrade so future readers don't think it was accidental.
