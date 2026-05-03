# sst/opencode #25602 — refactor(config+core): drop ConfigPaths.readFile, add AppFileSystem.readFileStringSafe, flatten TuiConfig.loadState

- PR: https://github.com/sst/opencode/pull/25602
- Author: kitlangton
- Head SHA: `57bc4f257065d5c03b1b9c4bc2abf4af99bbdade`
- Updated: 2026-05-03T14:38:28Z

## Summary
Three coupled refactors: (1) introduce `AppFileSystem.readFileStringSafe(path) → Effect<string | undefined>` that swallows `NotFound` (`packages/core/src/filesystem.ts:27,51-56,173`), (2) drop the now-redundant `ConfigPaths.readFile` helper (removed `loadFile` at the bottom of the old `tui.ts`), (3) inline the previously-Promise-based `loadFile`/`mergeFile`/`resolvePlugins` into the `loadState` Effect closure in `packages/opencode/src/cli/cmd/tui/config/tui.ts:71-156`, eliminating the `Effect.promise(() => mergeFile(...)).pipe(Effect.orDie)` bridge sprinkled across the four merge sites.

## Observations
- `packages/core/src/filesystem.ts:51-56`: `Effect.catchReason("PlatformError", "NotFound", () => Effect.succeed(undefined))` — confirm this is the canonical Effect-Platform shape for "file missing" today and not an older alias. If the upstream reason tag changes, this silently degrades to a thrown error. Worth a one-line comment that `NotFound` is the only error class intentionally swallowed; everything else (permission denied, IO error) should still defect via `orDie` at the call site.
- `packages/core/test/filesystem/filesystem.test.ts:68-94`: tests cover the happy path and the `NotFound` path. Missing: a "exists but unreadable" test (e.g. directory passed where file expected) to lock in that other PlatformErrors still propagate. Not blocking.
- `packages/opencode/src/cli/cmd/tui/config/tui.ts:91-100`: the inline `load` uses `Effect.catchCause` rather than `tapErrorCause + orElseSucceed`, with a comment explaining that `ConfigParse.jsonc/.schema` can sync-throw → defects. Good — that's the subtle bit. The `log.warn("invalid tui config", { path: configFilepath, cause })` keeps the existing observability.
- `packages/opencode/src/cli/cmd/tui/config/tui.ts:104-108`: `loadFile` calls `afs.readFileStringSafe(filepath).pipe(Effect.orDie)`. The `orDie` here means non-NotFound platform errors crash loadState rather than warn-and-skip. Previously, the `Effect.promise(() => mergeFile(...)).pipe(Effect.orDie)` had the same shape, so behavior is preserved. But: a malformed-permissions tui config used to also crash; with this refactor that's still true. Worth noting in the PR description so reviewers don't expect a behavior change.
- `packages/opencode/src/cli/cmd/tui/config/tui.ts:111-122`: `mergeFile` is now called as `yield* mergeFile(acc, file)` directly at the four merge sites (140, 147, 163). Cleaner and removes the `ctx` rebinding into `mergeFile` since `pluginScope(file, ctx)` now closes over the outer `ctx`.
- Diff is +large/-large but net negative once you account for removing `ConfigPaths.readFile`. Refactor is well-scoped and the test additions for `readFileStringSafe` justify the API extraction. Did not see a corresponding test update for `TuiConfig.loadState` itself — relying on existing config-loader tests to catch regressions. Acceptable since the behavior contract didn't change, only the Effect shape.
- One nit: `packages/opencode/src/cli/cmd/tui/config/tui.ts:228` (the trailing blank line after `runPromise((svc) => svc.get())`) — make sure the removed `loadFile` block didn't leave dead imports (`ConfigPaths.readFile` reference). Quick `grep ConfigPaths.readFile` across the package should return nothing.

## Verdict
`merge-after-nits`
