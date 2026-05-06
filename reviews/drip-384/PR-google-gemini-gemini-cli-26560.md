# google-gemini/gemini-cli#26560 — fix(core): handle invalid custom plans directory gracefully

- **Head SHA**: `dfedf22ae21c0206e5178a98531be69764b2893b`
- **Stats**: +48 / -1, 2 files

## Summary

When `planEnabled` is true and the user has set a custom `planSettings.directory` that fails validation (e.g., absolute path outside the project root, which `Storage.getPlansDir()` rejects), `Config.initialize()` would throw and brick the entire CLI startup. This PR catches the error from `getPlansDir()`, emits a `coreEvents.emitFeedback('warning', ...)` to surface the issue to the user, calls `setCustomPlansDir(undefined)` to clear the bad config, and re-resolves to the default project temp plans directory. CLI now starts cleanly with a visible warning instead of crashing.

## Specific citations

- `packages/core/src/config/config.ts:1447-1463`: wraps `this.storage.getPlansDir()` in a try/catch. The catch branch (a) extracts a safe error message via `error instanceof Error ? error.message : String(error)`, (b) emits `coreEvents.emitFeedback('warning', 'Invalid custom plans directory: <msg>. Falling back to default project temp directory.', error)` — three-arg call passes the original error as the third positional, preserving stack trace for downstream telemetry, (c) calls `this.storage.setCustomPlansDir(undefined)` to clear the bad value before re-resolving — correct ordering, otherwise the second `getPlansDir()` would throw the same error.
- `:1464`: after fallback, `plansDir = this.storage.getPlansDir()` re-resolves with the cleared custom dir. Implicit assumption: the default plan dir resolution can never throw — worth confirming in `Storage.getPlansDir()` source, but if it could, this would silently swallow a deeper bug.
- `packages/core/src/config/config.test.ts:4094-4123`: `it('should gracefully fallback to default plans directory if retrieving custom directory throw an error')` constructs `planSettings: {directory: '/outside/project/root'}`, asserts (a) `plansDir` contains `'plans'` (the default project temp shape) and does NOT contain `/outside/project/root`, (b) `coreEvents.emitFeedback` was called with `'warning'`, a message containing `'Invalid custom plans directory'`, and an `Error` instance, (c) the workspace context still adds the fallback plans dir. Three good axes covered.

## Verdict

**merge-after-nits**

## Rationale

Correct defensive fix for a real "user typo bricks the CLI" failure mode. The fallback chain is right (catch → emit warning → clear custom dir → re-resolve) and the test verifies all three observable side effects (path correctness, warning emission, workspace registration). Three nits: (1) the fallback assumes default `getPlansDir()` cannot throw — if a future refactor introduces validation on the default dir too (e.g., disk-full check), this catch becomes a silent-failure trap. A nested try/catch with an `if (error)` final-resort that disables plans entirely would be defense-in-depth, though arguably over-engineered for the common case. (2) `setCustomPlansDir(undefined)` mutates `Storage` state as a side effect of `Config.initialize()` — fine for the current code path but worth a comment naming why this can't simply pass an "ignoreCustom" flag to `getPlansDir()` (presumably because the custom dir is read elsewhere too and clearing it once is simpler). (3) The error message could be more actionable — "Invalid custom plans directory: <reason>" doesn't tell the user *which setting* to fix; appending "(check planSettings.directory in your config)" would shave a support cycle. None block merge.
