# sst/opencode#24783 — fix: add exit event fallback for child process close hang on Windows

- **PR**: https://github.com/sst/opencode/pull/24783
- **Author**: @bingkxu
- **Head SHA**: `fa9e317ac711662f749367c279b2974efe6de230`
- **Base**: `dev`
- **State**: OPEN
- **Scope**: +36 / -0 across 2 files (1 src + 1 test)

## Summary

On Windows, when a spawned child exits but inherits its stdout pipe down to a detached grandchild that keeps the pipe open, the parent never sees the `close` event because Node binds `close` to the lifetime of the pipe handles, not the lifetime of the child process. The current `cross-spawn-spawner.ts` implementation only resolves the `signal` Deferred from inside the `close` listener, so the `Effect.gen` block at `packages/core/src/cross-spawn-spawner.ts:271-285` hangs indefinitely on a clean child exit whenever any grandchild inherits the pipe. The fix adds a 2s fallback timer scheduled from inside the `exit` listener that resolves the same Deferred with the `exit` payload if `close` has not fired by then.

## Diff anchors

- `packages/core/src/cross-spawn-spawner.ts:273-285` — inside `proc.on("exit", ...)`, schedule `setTimeout(..., 2000)`. Inside the timer, re-check the `end` flag (set to `true` by the `close` handler) before flipping it and calling `Deferred.doneUnsafe(signal, Exit.succeed(args))`. The `if (!end)` guard at `:278` is load-bearing — without it, a `close` event arriving inside the 2s window would trigger a double-resolve panic on the Deferred.
- `packages/core/test/effect/cross-spawn-spawner.test.ts:285-313` — Windows-only test gated by `process.platform !== "win32"` early-return. Spawns a child that itself spawns a detached grandchild inheriting `stdio: ['ignore', process.stdout, 'ignore']` and runs for 10s, then `process.exit(0)`s. The bound check `expect(elapsed).toBeGreaterThan(1500)` at `:309` proves the fallback was the resolver (not `close` firing early through some other path), and `< 5000` at `:310` allows for spawn overhead without giving up the regression-protection signal.

## What I'd push back on (nits)

1. **2000ms is a magic number**. Extract to `const EXIT_FALLBACK_MS = 2000` near the top of the file with a comment naming the Windows-grandchild-pipe-inheritance failure mode. Future tuning happens in one place; reviewers don't have to re-discover the rationale.
2. **The fallback is unconditional, not Windows-gated.** On macOS/Linux, where `close` reliably fires after `exit`, the timer adds 2s of `setTimeout` keeping a no-op alive on every spawned process. This is harmless functionally but it does keep the event loop active and is cheap-but-pointless work. Either gate on `process.platform === "win32"` (matches the test gate exactly), or document explicitly why the unconditional shape is preferred.
3. **No `clearTimeout` in the `close` path.** When `close` does fire normally before 2s, the `setTimeout` callback still runs, takes the `if (!end)` early-exit, and is silent. Functionally fine, but a `clearTimeout(fallback)` inside `proc.on("close", ...)` makes the intent visible and saves the event-loop tick.
4. **`exit` payload semantics on Windows**. The `args` captured by the `exit` listener at `:274` is `[code, signal]`. On the macOS/Linux happy path, `close` is called with the same shape and the resolved Exit value matches what callers see today. On the Windows fallback path, the `Exit.succeed(args)` resolves with the `exit` event's `args` — verify via grep that no downstream consumer of `signal` discriminates between "exit-payload" and "close-payload" shapes (they should be identical in `child_process` types, but a one-line confirmation in the PR body would close the question).

## Verdict

**merge-after-nits** — fix is correctly diagnosed (the inherited-pipe-keeps-`close`-pending failure mode is a known real Windows behavior), the placement is right (inside `exit` rather than a top-level timeout that would race spawn errors), and the test is unusually principled with both lower and upper time bounds. The `EXIT_FALLBACK_MS` extraction and `clearTimeout` cleanup are both two-line changes worth folding in before merge.

Repo coverage: sst/opencode (TUI/desktop runtime).
