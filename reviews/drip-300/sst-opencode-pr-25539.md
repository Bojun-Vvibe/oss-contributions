# sst/opencode PR #25539 — refactor(cli/github+run): Stage 4 — drop AppRuntime.runPromise bridges

- Author: kitlangton
- Head SHA: `86c443891001d9a5feec90525418891ecf08fe9d`
- Diff: +28 / -26 across 2 files
- Files: `packages/opencode/src/cli/cmd/github.ts`, `packages/opencode/src/cli/cmd/run.ts`

## Observations

1. **`github.ts:206-211` and `run.ts:300` hoist services to the Effect handler scope**: e.g. `const modelsDev = yield* ModelsDev.Service`, `const gitSvc = yield* Git.Service`, `const agentSvc = yield* Agent.Service`. The pattern is now uniform across both files — services are resolved once at handler entry, then closures inside `Effect.promise(async () => …)` use the captured handle. Clean and readable.
2. **`github.ts:213-214` removes `AppRuntime.runPromise(ModelsDev.Service.use(s => s.get()))` in favor of `Effect.runPromise(modelsDev.get())`** — same wire effect, no double-runtime. This is the structural payoff of Stage 4: no nested runtime boundaries. The downstream `delete p[...]` provider-pruning step is preserved verbatim.
3. **`github.ts:262-265` collapses the nested `Git.Service.use((git) => git.run(…))` into `gitSvc.run(…)`** — three call sites in `gitText`/`gitRun`/`gitStatus` (`github.ts:508-525`) all migrate consistently. Watch for: the `Process.RunFailedError` is still thrown synchronously inside the `async` closures, which is fine, but the migration didn't change the error path so behavior is unchanged.
4. **`github.ts:946-957` for `chat()`**: the change from `AppRuntime.runPromise(Effect.gen(function*() { const prompt = yield* SessionPrompt.Service; ... }))` to using the captured `sessionPrompt` is functionally identical, but the new code still wraps in `Effect.gen` + `Effect.runPromise` which now does no service resolution at all — minor opportunity to simplify further by direct `Effect.runPromise(sessionPrompt.prompt({...}))`, though acceptable as-is for diff minimality.
5. **No tests touched** — refactor relies on existing `cli/cmd/github` integration tests for safety. Given the surface (CLI subcommand wiring) and the mechanical 1:1 substitution, that's defensible. A quick smoke run of `opencode github run --token mock --event ...` end-to-end would still be reassuring.
6. **`run.ts:602`** now does `await Effect.runPromise(agentSvc.get(name))` directly. Captured handle pattern matches `github.ts`. Diff is minimal and self-consistent.

## Verdict: `merge-as-is`

This is exactly the kind of mechanical Stage-N refactor that benefits from being landed quickly while the call sites are still fresh in everyone's head. The pattern is consistent, no behavior should change, and the parent service-extraction work (Stages 1-3) presumably already de-risked the runtime semantics.
