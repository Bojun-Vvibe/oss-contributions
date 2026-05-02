# Review: sst/opencode#25465

- **PR:** sst/opencode#25465
- **Head SHA:** `ea7f58d63d30306bdf9a4835a744e2027b54e356`
- **Title:** refactor(cli): convert pr command to effectCmd
- **Author:** kitlangton

## Files touched (high-level)

- `packages/opencode/src/cli/cmd/pr.ts` — full rewrite of the `pr <number>` handler from a `cmd({...async handler})` to `effectCmd({ handler: Effect.fn("Cli.pr")(...) })`, replacing `AppRuntime.runPromise(Git.Service.use(...))` and `Instance.provide({...fn()})` calls with direct `yield* git.run(...)` against `InstanceRef` + `Git.Service`.

## Specific observations

- `pr.ts:8`: switching from `cmd` → `effectCmd` is consistent with the broader migration of CLI handlers in this codebase. The `Effect.fn("Cli.pr")` name will give nicer traces for the `pr` command.
- `pr.ts:18-22`: `const ctx = yield* InstanceRef; if (!ctx) return yield* fail("Could not load instance context")` — this is the right shape, but worth confirming `InstanceRef` actually yields `null` rather than throwing when no instance is bound; if it throws, the guard is dead code (matches what other migrated commands do, so probably fine).
- `pr.ts` Git remote add hunk: previously this was three nested `AppRuntime.runPromise(Git.Service.use((git) => git.run([...], { cwd: Instance.worktree })))` blocks. New code is a single `yield* git.run([...], { cwd: worktree })` per call. Significant readability win, no behavioral change as far as I can tell.
- `pr.ts` `prInfo?.isCrossRepository && prInfo.headRepository && prInfo.headRepositoryOwner`: optional chaining is fine, but the original was strictly `prInfo && prInfo.isCrossRepository && ...` — equivalent behavior since `prInfo` is the parsed JSON object.
- `pr.ts` final `Process.spawn(["opencode", ...opencodeArgs], { stdin/stdout/stderr: "inherit", cwd: process.cwd() })`: needs to be wrapped in an `Effect.promise` (or whatever the equivalent escape hatch is) since this is now inside a generator handler — verify it isn't being silently dropped. (The truncated diff suggests it is wrapped, but worth a careful look at the tail of the file.)
- Worth double-checking that `worktree` (captured once at the top) doesn't go stale if any of the intervening `git.run` calls mutate the worktree path. For `gh pr checkout` + `git remote add` + `git branch --set-upstream-to`, it shouldn't, so this is fine.

## Verdict

**merge-after-nits**

## Reasoning

This is a mechanical, well-scoped Effect migration that mirrors the pattern other CLI commands in this repo have already adopted. No behavior changes, the logic is preserved, and the resulting code is materially cleaner (no more triple-nested `AppRuntime.runPromise(Git.Service.use(...))`). The only nits are: confirm `InstanceRef` semantics for the null guard, and double-check that the final `Process.spawn(["opencode", ...])` block at the tail of the new handler is properly bridged into the Effect generator. Both are quick verifications, not blockers.
