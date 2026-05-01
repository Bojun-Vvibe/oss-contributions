# sst/opencode#25029 — refactor(opencode): Faster cold starts using dynamic import splitting

- **PR**: https://github.com/sst/opencode/pull/25029
- **Head SHA**: `7ac34f9edf34f54939314c71b4897f251fbaae64`
- **Files**: ~85 files; +6586/-6309. Each subcommand under `packages/opencode/src/cli/cmd/<name>.ts` split into `<name>/command.ts` (yargs definition, dynamic-import handler) and `<name>/handler.ts` (the actual logic).
- **Verdict**: **merge-after-nits**

## Context

CLI cold start was dominated by static-import tree resolution since every subcommand was eagerly resolved at top-level even if the user only ran `opencode --version`. PR splits each subcommand into a tiny `command.ts` (yargs definition with a `handler: async (args) => { await import("./handler").then(({ ...Handler }) => Handler(args)) }` shim) plus `handler.ts` (the original implementation, unchanged module-internal). PR body cites `opencode --version` time dropping from `0.737s` total → `0.183s` (~75% reduction) and `opencode debug` from `0.765s` → `0.194s`; even the heavy `opencode run` path drops from `0.779s` → `0.434s`.

## What's right

- **Pattern is uniform across ~20 subcommands.** Every split follows the same shape: `command.ts` exports `cmd({ command, builder, handler: async (args) => (await import("./handler")).<Name>Handler(args) })`. See `account/command.ts:11-14`, `acp/command.ts`, `agent/command.ts`, etc. Reviewers can confirm the pattern by spot-checking 3-4 splits.
- **Internal helpers correctly extracted, not duplicated.** `account/format.ts` (new at `:1-21`) lifts the previously-private `dim`, `activeSuffix`, `formatAccountLabel`, `formatOrgChoiceLabel`, `formatOrgLine` out of the handler so the command.ts shim doesn't pay handler's import cost. The handler.ts now imports `formatAccountLabel, formatOrgChoiceLabel, formatOrgLine` from `./format`. Correct discipline — the format helpers are pure, no dynamic-import overhead, fast to load.
- **Tests updated mechanically with import-path rewrites.** `test/cli/account.test.ts`, `github-action.test.ts`, `github-remote.test.ts`, `import.test.ts`, `plugin-auth-picker.test.ts`, `tui/thread.test.ts`, `fixture/plug-worker.ts`, `plugin/install.test.ts` each get a one-line import-path update. Mechanical sweep, low risk.
- **Subcommands committed separately per PR body.** Author explicitly states "I've split the commits by subcommand to make review easier. It should be simple for an agent to review the changes commit-by-commit to confirm the code is identical before and after." Correct discipline for a 6586/6309-line refactor.
- **`packages/opencode/src/index.ts` (+28/-26)** updates the top-level command registry to import the lighter `command.ts` modules, not the full handlers. This is the load-bearing change that actually realizes the speedup.

## Risks / nits

- **Dynamic `import("./handler")` swallows errors silently if the handler module fails to load.** The shim shape `handler: async (args) => { await import("./handler").then(({ <Handler> }) => <Handler>(args)) }` will surface a module-load error as a yargs-handler rejection, but the user-visible message may be less helpful than the prior eager-import error (which would have surfaced at process start). Recommend a one-line wrapper that catches `import()` failures and prints a clear "Failed to load subcommand <name>: <error>" message before re-throwing.
- **No benchmark regression test in CI.** PR body shows compelling local benchmarks but no CI guard to prevent regression (e.g., a future PR re-introducing a top-level `import` of `./run/handler` from `index.ts`). Recommend a smoke test asserting `opencode --version` finishes in under N ms (with N = 2-3× the post-fix number to allow CI variance) so a regression is caught at PR time.
- **`packages/opencode/src/cli/cmd/debug/handler.ts` (+358) consolidates 9 prior debug submodule files** (`agent.ts`, `config.ts`, `file.ts`, `index.ts`, `lsp.ts`, `ripgrep.ts`, `scrap.ts`, `skill.ts`, `snapshot.ts`, `startup.ts`) into a single handler. This is a structural change beyond pure refactor — the debug submodules are no longer independently lazy-loadable. If `debug ripgrep` is the only common path, it now pays the cost of loading `debug snapshot`'s deps too. Recommend documenting the choice ("debug subcommands are co-located because they share <reason>") or splitting them into per-subcommand handlers like the other splits.
- **`packages/opencode/src/cli/cmd/github/handler.ts` (+1586)** is a single 1586-line handler. Same observation — at this size the lazy-load benefit is reduced because users invoking any `opencode github *` subcommand pay the full cost. Could split into `github/<subcommand>/handler.ts` for finer-grained laziness, though that's a follow-up.
- **`packages/opencode/src/cli/network.ts` (+3/-2)** and `packages/opencode/src/temporary.ts` (+1/-1) — small drive-by changes that should ideally be in their own PR or called out in the body. Minor.

## Verdict

**merge-after-nits.** Real, measurable cold-start win (~75% on the lightweight paths) via mechanically-applied dynamic-import splitting. The pattern is uniform and reviewable commit-by-commit per the author's own discipline. Strongest nits: (1) a CI benchmark guard so this doesn't regress, (2) cleaner error path for `import()` failures so users see actionable messages, (3) document or split the consolidated `debug/handler.ts` and `github/handler.ts` since they're large enough to defeat some of the laziness win.
