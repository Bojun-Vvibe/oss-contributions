# sst/opencode PR #25507 — feat(cli): add `instance: false` opt-out to effectCmd

- **Repo:** sst/opencode
- **PR:** #25507
- **Head SHA:** `7d650743da3042582db393c6d0662925975fe518`
- **Author:** kitlangton
- **Title:** feat(cli): add instance: false opt-out to effectCmd
- **Diff size:** +45 / -17 across 2 files
- **Drip:** drip-293

## Files changed

- `packages/opencode/src/cli/cmd/serve.ts` (+~14/-~10) — converts `ServeCommand` from the legacy `cmd()` factory to `effectCmd({ instance: false, handler: Effect.fn(...) })`, threads `Effect.promise` around `resolveNetworkOptions`, `Server.listen`, the never-resolving keep-alive, and `server.stop`.
- `packages/opencode/src/cli/effect-cmd.ts` (+~31/-~7) — extracts an `EffectCmdOpts<Args, A>` interface, adds an `instance?: boolean` knob with extensive doc comment, and short-circuits the yargs handler when `opts.instance === false` to skip `InstanceStore.Service.use(...)` entirely.

## Specific observations

- `effect-cmd.ts:42-46` — JSDoc explicitly enumerates the safe-to-skip commands (`models`, `serve`, `web`, `account`, `db`, `upgrade`); good guidance, but it's worth noting that *only* `serve` is migrated in this PR — the rest still pay the `InstanceBootstrap` cost. Either land the rest in follow-ups (a TODO referencing the issue would help) or admit the list is aspirational.
- `effect-cmd.ts:75-78` — the early-return path runs the handler under bare `AppRuntime.runPromise(opts.handler(args))` without any `Effect.ensuring(...)` cleanup. The legacy path has `store.dispose(ctx)` on every Exit. For `serve` this is fine (process-lifetime daemon), but the doc comment should warn future authors who pick `instance: false` for shorter-lived commands that they own their own resource cleanup.
- `cmd/serve.ts:16-25` — `Effect.fn("Cli.serve")(function* (args) { ... })` is the right shape, and the explanatory comment at `serve.ts:10-12` ("Server loads instances per-request via x-opencode-directory header") is exactly the justification reviewers need. Keep it.
- `cmd/serve.ts:21` — wrapping `new Promise<void>(() => {})` in `Effect.promise` is correct but reads oddly; consider `Effect.never` (effect/Effect) instead, which is the canonical "block forever" primitive and avoids the unused promise allocation. Not blocking.
- The `EffectCmdOpts` interface is defined at module scope but never exported — fine for now, but if you intend to migrate more commands they'll want access to compose the type. Mark as `export interface` proactively.

## Verdict: `merge-after-nits`

The opt-out shape is right and the JSDoc is unusually thorough. Two ergonomic nits (`Effect.never` over the no-op promise; export `EffectCmdOpts`) plus one doc clarification (the listed safe-to-skip commands aren't all migrated yet) would land this cleanly.
