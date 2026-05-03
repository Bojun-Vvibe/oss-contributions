# sst/opencode PR #25537 — refactor(cli/providers): flatten — Effect-native handlers end-to-end

- Author: kitlangton
- Head SHA: `eb4a90eb2a42f1c5e383010ae6fca11b0af4b45f`
- Diff: +189 / -202, single file `packages/opencode/src/cli/cmd/providers.ts`
- Stacked on #25532; auto-merge enabled (squash)

## Observations

1. **`providers.ts:8-10` deletes the top-level `getModels` / `refreshModels` helpers** that wrapped `AppRuntime.runPromise(ModelsDev.Service.use(...))`. Handlers now `yield* ModelsDev.Service` and call `.get()` / `.refresh(true)` directly. This is the correct end-state — top-level Promise bridges were the last vestige of the pre-Effect handler shape, and they made the call graph unnecessarily deep.
2. **`providers.ts:241-283` (ProvidersListCommand)** drops the outer `Effect.promise(async () => {...})` wrap and converts inner `await Effect.runPromise(authSvc.all())` to `yield* Effect.orDie(authSvc.all())`. The `Effect.orDie` is the right escape hatch where the typed error channel is intentionally being collapsed to a defect at the CLI surface — same observable behavior (uncaught throw → Bun process exit) as the old `runPromise` style, just typed correctly.
3. **`providers.ts:303+` (ProvidersLoginCommand)** — same shape: outer wrap removed, individual `await prompts.X()` and `await fetch()` calls now sit inside `yield* Effect.promise(() => ...)` blocks. PR description claims "Same observable behavior — same prompts, same exit codes, same stderr text." Worth a snapshot of the `--help` and an interactive `providers login` session to verify, but the mechanical conversion looks correct from the diff.
4. **PR description claims "8 → 1 `AppRuntime.runPromise` call"**, with the remaining one being the `put` helper used by the still-Promise-style `handlePluginAuth`. That's a reasonable scope boundary — `handlePluginAuth` likely lives in another module and converting it would balloon the diff. A `// TODO: convert once handlePluginAuth is Effect-native` comment on the surviving `put` would help future cleanup.
5. **`UI.CancelledError` conversion** — `throw new UI.CancelledError()` becoming `yield* Effect.die(new UI.CancelledError())` is the right call (defect, not failure) since cancellation is a process-level signal in this codebase, not a recoverable typed error.

## Verdict: `merge-after-nits`

Mechanical, well-scoped Effect-native flatten that lines up with the broader Stage 4 series. Pre-merge: confirm the manual smoke (list + login) actually was run, and consider a one-line `// TODO` on the surviving `put` helper to flag it as the next conversion target. Auto-merge being on means review velocity matters here — these refactor PRs land in stacks.
