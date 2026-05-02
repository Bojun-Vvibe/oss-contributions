# sst/opencode PR #25485 — refactor(cli): convert debug agent command to effectCmd

- Head SHA: `bbbe611d5ba049f4da388fe16d770cdf904acaa4`
- URL: https://github.com/sst/opencode/pull/25485
- Size: +89 / -100, 1 file
- Verdict: **merge-after-nits**

## What changes

`packages/opencode/src/cli/cmd/debug/agent.ts` is migrated from the legacy
`cmd(...)` + `bootstrap(...)` style to the new `effectCmd` runtime. Handler
becomes `Effect.fn(...)` that pulls `InstanceRef` and `InstanceStore.Service`,
then `Effect.ensuring(store.dispose(ctx))` cleans up the instance context.
`getAvailableTools` and `createToolContext` are also rewritten as Effect
generators, and `resolveTools` becomes synchronous.

## What looks good

- The `Effect.ensuring(store.dispose(ctx))` pattern matches the auto-dispose
  contract introduced in #25481 — instance lifetime is now scoped to the
  command invocation rather than relying on `bootstrap` ambient cleanup.
- `process.exit(1)` is correctly replaced with `yield* fail("", 1)` so the
  Effect runtime gets a chance to run finalizers (this was a real footgun in
  the old version — `process.exit` would skip cleanup).
- `resolveTools` going from `async` to sync is a nice cleanup: the previous
  `await` was spurious since `Permission.disabled` is sync.

## Nits

1. `if (!ctx) return` (around line 38 of the new handler) silently
   short-circuits when there is no `InstanceRef`. The legacy code would
   `bootstrap(process.cwd(), ...)` and proceed. Worth at least logging
   `"agent command requires an instance context"` to stderr — otherwise a
   user invoking `debug agent` outside a project just sees nothing happen.
2. The new `getAvailableTools` Effect.fn drops the explicit `AppRuntime.runPromise`
   wrapper but is now executed inside the outer `Effect.fn("Cli.debug.agent.body")`.
   Confirm the tracing span name `Cli.debug.agent.getAvailableTools` is
   intentional for observability dashboards (the old version had no span
   here at all, so this is strictly an upgrade — just call it out in the
   PR body so reviewers know the tracing surface grew).
3. `parseToolParams(args.params)` (line ~62) now passes `args.params`
   directly without the `as string | undefined` cast that existed before.
   Type narrowing relies on yargs' inferred type. If `--params` ever
   becomes optional with a default value this could silently coerce.

## Risk

Low — pure CLI plumbing with no behavior change beyond cleanup ordering.
The `Effect.ensuring` makes resource cleanup *more* reliable than before.
