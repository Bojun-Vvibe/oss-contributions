# sst/opencode PR #25479 — Convert debug subcommands to effectCmd

- Head SHA: `8331cab632bf619a49aa2cd5c52f666e41ed3264`
- URL: https://github.com/sst/opencode/pull/25479
- Size: +185 / -139, 6 files (`packages/opencode/src/cli/cmd/debug/{config,file,lsp,ripgrep,skill,snapshot}.ts`)
- Verdict: **merge-after-nits**

## What changes

Continuation of the `effectCmd` migration (drip-285/286/287 covered other
CLI surfaces). All `debug` subcommands — `config`, `file
search/read/status/list/tree`, `lsp diagnostics/symbols/document-symbols`,
`rg tree/files/search`, `skill`, `snapshot track/patch/diff` — are
rewritten from the legacy `cmd({ async handler() { await
bootstrap(process.cwd(), async () => { await AppRuntime.runPromise(...) }) } })`
shape to `effectCmd({ handler: Effect.fn(...)(function* () { ... }) })`
that pulls `InstanceRef` + `InstanceStore.Service` from the runtime and
ties each handler's lifetime to `Effect.ensuring(store.dispose(ctx))`.

`Instance.directory` accesses (e.g. ripgrep.ts:303-304, 332) are
replaced with `ctx.directory` from the `InstanceRef` — eliminates
implicit reliance on the `Instance` global inside CLI commands.

## What looks good

- The `if (!ctx) return` guard (every handler, e.g. config.ts:26,
  file.ts:66, lsp.ts:212) is consistent across all 6 files. Previously
  the `bootstrap` wrapper enforced "must be in a project" implicitly;
  the new version makes it explicit and uniform.
- `Effect.ensuring(store.dispose(ctx))` (e.g. file.ts:73, lsp.ts:224,
  rg.ts:315) is the right place to put cleanup — it runs on success,
  failure, *and* interrupt, so a Ctrl-C while `lsp diagnostics` is
  blocked on `touchFile` no longer leaks the instance handle.
- `Effect.fn("Cli.debug.lsp.diagnostics")` etc. give every command a
  named span — a real observability win over the previous
  `runPromise(...)` inside an anonymous `bootstrap` callback.
- ripgrep.ts:311 / 360 / 386 wrap the underlying `Ripgrep.Service.use`
  in `Effect.orDie` because `rg` failures are programmer errors at the
  CLI layer; previously these were silently swallowed by `runPromise`.
  Strictly better.

## Nits

1. `lsp.ts:240` and `lsp.ts:262` keep `using _ = Log.Default.time(...)`
   inside `Effect.gen`. `using` works because Effect's generator runtime
   currently honors `Symbol.dispose` on locals, but this is a fragile
   contract — if the timer needs to span the `Effect.ensuring` boundary
   (which now holds the dispose), a `Effect.acquireRelease` or
   `Effect.scoped` pattern would be more honest about lifetime. Worth
   a follow-up issue, not a blocker.
2. The `if (!ctx) return` early-return (everywhere) silently exits with
   code 0 when there's no instance context. The legacy `bootstrap` path
   would have surfaced the "no project" failure mode. At minimum,
   write a `process.stderr` line ("debug commands require a project
   directory") so users don't silently get empty output and assume
   their query matched nothing.
3. file.ts now imports `cmd` (line 48) but only uses it for the
   parent `FileCommand` group at line 173 — the per-leaf commands all
   moved to `effectCmd`. That's correct, but a comment near line 48
   ("parent group still uses cmd; only leaves are effectCmd") would
   save the next reader a `git blame`.
4. snapshot.ts:472 / 504 / 529 use `console.log(out)` instead of
   `process.stdout.write(out + EOL)` like the rest of the file. Minor
   inconsistency — `console.log` will JSON-stringify objects with the
   default depth limit and append a stray newline; the other commands
   went out of their way to control formatting.

## Risk

Low/medium. This is a 6-file behavioral refactor, not a pure rename —
the lifetime model for `InstanceStore.dispose` is now per-CLI-handler
instead of bootstrap-scoped, so any handler that *forgets* to take
`store.dispose(ctx)` in its `Effect.ensuring` will leak. All 12
handlers in this PR do take it, but the pattern is copy-paste and the
type system doesn't enforce it. A small `withInstanceLifecycle` helper
that takes `(ctx, body)` and returns the wrapped Effect would close
that footgun in a follow-up.

The behavior under interrupt is genuinely improved — the old
`bootstrap(...)` swallowed interrupt signals at the Promise boundary;
the new `Effect.ensuring` chain propagates them.
