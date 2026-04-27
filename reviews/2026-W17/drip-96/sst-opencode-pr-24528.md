# sst/opencode #24528 — fix(tui): startup rejection handling

- Author: simonklee
- Head SHA: `c3f867144902c704c5905c04570835eb6565a659`
- +101 / −71 across `packages/opencode/src/cli/cmd/tui/app.tsx` (+72/-71), `packages/opencode/test/cli/tui/app.test.ts` (+29 new)
- PR link: <https://github.com/sst/opencode/pull/24528>

## Specifics

- Root cause is the `new Promise<void>(async (resolve) => { … })` async-executor anti-pattern at the old `app.tsx:117`: any throw from `createCliRenderer(rendererConfig(input.config))` or `renderer.waitForThemeMode(1000)` was swallowed by the executor's promise-never-resolves, so the CLI hung forever instead of surfacing renderer init failures (`setRawMode failed with errno: 9` is the canonical example, asserted in the new test). The `oxlint-disable-next-line no-async-promise-executor -- intentional` pragma was the smoking gun — that lint exists precisely for this class of bug.
- New shape at `app.tsx:117-201` is `new Promise<void>((resolve, reject) => { … void (async () => { … })().catch(reject) })`. The IIFE wrapping is the standard fix: the executor itself stays synchronous, the async work runs as a normal promise chain, and any rejection (sync throw or async failure) routes through `.catch(reject)`. Resolve still happens once the renderer-bound `render(...)` call returns, matching the prior contract.
- `win32DisableProcessedInput()` moved from immediately-after-`unguard()` (old `app.tsx:120`) to the *first* line inside the new IIFE (`app.tsx:131`). Behaviorally equivalent for the happy path, but if the IIFE rejects after this line, `win32InstallCtrlCGuard`'s `unguard?.()` is no longer called as part of cleanup — `unguard` only runs in `onExit`, which the caller invokes on graceful exit but won't invoke on a startup rejection. Worth checking whether the caller's `.catch` path runs `unguard`/`onExit`, or whether `win32` users now leak a Ctrl-C guard if renderer init fails.
- Test at `app.test.ts:8-29` mocks `@opentui/core`'s `createCliRenderer` to reject with the canonical `setRawMode failed with errno: 9` and asserts the returned promise rejects with that exact error within 100ms (`Promise.race` against `Bun.sleep(100)`). The 100ms ceiling locks down the "not hanging anymore" property — without the race, a regression that re-introduced the swallow would hang the test runner instead of failing fast. Good test design.
- The 71/72 line counts are dominated by re-indentation inside the IIFE (the entire `<ErrorBoundary>` provider tree shifted right by 2 spaces). Pure mechanical change; no JSX semantics changed.

## Concerns

- The `onExit` and `onBeforeExit` closures defined at `app.tsx:124-130` are not invoked on the new rejection path — only on the normal exit/error-boundary path. If renderer init fails partway through (e.g. after some terminal-mode side effect), terminal state may be left in an inconsistent mode (`win32DisableProcessedInput` already ran, raw-mode partially flipped, etc.). The PR description claims "destroy any partially initialized renderer before rejecting to restore terminal state" but I don't see a `renderer?.destroy()` or equivalent in the rejection path — the IIFE rejects whatever `createCliRenderer`/`render` throws and that's it. Either the description overstates the fix or the destroy is implicit in opentui's failure semantics; worth confirming.
- The test only covers `createCliRenderer` rejection, not `waitForThemeMode` rejection or `render` rejection. The 1000ms `waitForThemeMode` path has its own failure modes that the new shape now surfaces — adding one more test for a `waitForThemeMode`-throw or `render`-throw would lock down all three async hops in the IIFE.
- Resolution semantics: `render(...)` is awaited inside the IIFE but `resolve()` is *never* explicitly called — the original code also never called `resolve()` after `await render(...)`, so the promise only settled via `onExit → resolve`/process exit. With the IIFE pattern, the same is true: a successful render returns from the IIFE and the outer Promise stays pending until `onExit` (or process termination). That's preserved behavior, but it means the promise-shape is a "lifecycle handle" not a "startup handle" — not a regression, just worth noting that this PR doesn't fix the still-pending semantics, only the throw-on-startup case.

## Verdict

`merge-after-nits` — the async-executor anti-pattern fix is correct and the test is well-designed (the 100ms race is the right way to assert "doesn't hang"). Two follow-ups: (1) confirm the partial-renderer-cleanup claim from the PR body actually has code behind it, and (2) extend test coverage to `waitForThemeMode`/`render` rejection branches to lock down all three async hops in the IIFE.
