# sst/opencode #25672 — fix: prevent pkill hang when close event never fires

- SHA: `f3ed12bdbbb0687090d43d1d29f11bf4ab5c6b02`
- State: OPEN, +10/-3 across 2 files
- Files: `packages/core/src/cross-spawn-spawner.ts`, `packages/opencode/src/tool/shell.ts`

## Summary

Three correlated fixes for a hang when `pkill -f` orphans children that keep stdio pipes open: (1) resolve the exit-signal `Deferred` from the `exit` event instead of `close`, (2) drop `Deferred.await(signal)` from the SIGKILL escalation path, (3) add an `Effect.catch` around `handle.exitCode` inside the bash tool's `raceAll` so an Errored exit no longer poisons the race.

## Notes

- `packages/core/src/cross-spawn-spawner.ts:273-280` — moves `Deferred.doneUnsafe(signal, ...)` into the `exit` listener guarded by `!end`. This is the load-bearing fix: Node guarantees `exit` fires when the child process terminates even when child-of-child holds pipe FDs, while `close` waits for all stdio FDs to drain. Correct. Worth confirming the existing `close` handler at line ~283 still cleans up listeners and does not double-resolve (the `if (end) return` guard appears to handle this).
- `packages/core/src/cross-spawn-spawner.ts:397, 434` — replacing `send("SIGKILL").pipe(Effect.andThen(Deferred.await(signal)), Effect.asVoid)` with bare `send("SIGKILL")` is justified by the PR body ("SIGKILL is synchronous on Linux"), but on macOS/BSD the kernel still requires the parent to reap the zombie before `wait()` returns. The previous `Deferred.await` also handled the case where another caller had already triggered termination. Net behavior on macOS deserves a sentence in the PR description; on Linux the change is safe given fix (1).
- `packages/opencode/src/tool/shell.ts:516-519` — `Effect.catch(() => Effect.succeed({ kind: "exit" as const, code: null }))` swallows *any* failure from `handle.exitCode`, including non-termination errors (e.g., spawn failures surfaced late). Consider `Effect.catchAll(() => ...)` or filtering on a specific tag so genuine bugs are not silently coerced into a clean exit. At minimum, log at debug level when this branch fires.
- No tests. The reproduction (`pkill -f vim` from the TUI) is fragile to automate, but a unit test around `cross-spawn-spawner` that spawns a child which holds an open pipe FD via a grandchild would lock fix (1) into the suite. Without a test, the next refactor that "tidies up" the duplicated `end = true` block at lines 273-280 vs. the `close` handler will silently regress this.

## Verdict

`merge-after-nits` — fix (1) is unambiguously correct and resolves a real production hang. Tighten the `Effect.catch` in `shell.ts` to avoid swallowing unrelated failures, add a one-line note about macOS SIGKILL behavior, and a regression test would be welcome but not blocking.
