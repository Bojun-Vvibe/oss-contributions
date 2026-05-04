# sst/opencode #25693 — Improve shell completion docs and avoid full init for completion probes

- **Head SHA reviewed:** `caf7e978bd578fd6238a504c8b844431bbe81930`
- **Author:** luojiyin1987
- **Size:** +93 / -1 across 8 files
- **Verdict:** merge-after-nits

## Summary

Two-pronged improvement to shell completion: (1) add user-facing docs in
`README.md`, `README.zh.md`, and the web docs (`packages/web/src/content/docs/index.mdx`,
`zh-cn/index.mdx`) for `opencode completion`; (2) introduce
`isShellCompletionInvocation()` in `packages/opencode/src/cli/completion.ts`
and use it in `src/index.ts` and `src/provider/models.ts` to short-circuit
expensive startup work (config preflight, models cache refresh) when the
process is being invoked purely to emit completion candidates.

## What I checked

- `packages/opencode/src/cli/completion.ts:1-4` — the predicate is two
  cheap tests: `--get-yargs-completions` flag (yargs probe) OR first
  non-flag positional being `completion` (the user-typed install command).
  Logic matches the existing yargs completion contract.
- `packages/opencode/src/index.ts:42,58,93` — invocation pre-evaluated
  once into `shellCompletionInvocation` and then used inside the yargs
  middleware to bail before mutating `process.env.OPENCODE_PURE` etc.
  Good: avoids reading args twice and keeps the effect predictable.
- `packages/opencode/src/provider/models.ts:184` — replaces the
  ad-hoc `process.argv.includes("--get-yargs-completions")` guard with
  the shared helper, so the `Schedule.spaced("60 minutes")` background
  refresh doesn't fork during completion probes. Correct, and removes
  the duplicated literal.
- `packages/opencode/test/cli/completion.test.ts:1-19` — tests cover the
  three relevant axes: typed `completion` subcommand (with and without
  flags), the yargs probe flag, and three negative cases (empty args,
  normal `run` command, and the false-positive `./completion`). Good
  guard against future drift.
- README/docs additions are stylistically consistent with the surrounding
  install instructions and the macOS `~/.bash_profile`/`~/.zprofile`
  hint is genuinely useful.

## Nits

1. `completion.ts:3` — `argv.find((arg) => !arg.startsWith("-")) === "completion"`
   walks the whole array on a miss; for the common no-args path it's
   one comparison, so this is fine, but a `for…of` early-return reads
   slightly cleaner and avoids constructing the closure on the hot
   startup path.
2. The two README blocks duplicate the install snippet verbatim. Not a
   blocker — the docs site uses a separate MDX file anyway — but a future
   refactor could DRY this via an include.
3. `index.ts` — `shellCompletionInvocation` is computed at module top
   level and referenced inside the middleware. That's intentional (so
   the middleware closure is cheap), but a one-line comment above the
   `const` would help future readers understand why it isn't recomputed
   inside the middleware.

## Risk

Low. The new helper is pure, well-tested, and the call sites only
*skip* work — they don't change behavior for normal invocations. Worst
case is a regression where a typed argument that happens to literally
equal `"completion"` (e.g. a session name) bypasses startup work; the
test at `completion.test.ts:15` shows the author considered this
(`"./completion"` returns false), and `["run", "completion"]` would
correctly resolve to `run` first.

## Recommendation

Approve after the optional one-line comment in `index.ts`. The change
is a clear quality-of-life win for shell users and removes a real
startup cost for completion probes.
