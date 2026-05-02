# google-gemini/gemini-cli PR #26367 — fix(cli): print --version on real stdout before patchStdio

- Repo: `google-gemini/gemini-cli`
- PR: #26367
- Head SHA: `3f69fde5ac9f705b459fd4df354162353face484`
- Author: Ritaldojr7

## Summary
`patchStdio` redirects `process.stdout.write` into the coreEvents bus, so
yargs `--version` (which prints via `process.stdout.write`) emitted nothing
on real stdout. The nightly verify-release smoke test (`$(gemini
--version)` compared against the published version) was failing because
the capture was empty. This PR short-circuits `--version` / `-v` on real
stdout before `patchStdio` runs. Fixes #26359.

## Specific references
- `packages/cli/src/gemini.tsx:38` — adds `getVersion` to the
  `@google/gemini-cli-core` import (uses the same module that yargs
  ultimately reads from, ensuring no version drift between the two paths).
- `packages/cli/src/gemini.tsx:41` — adds `import { hideBin } from
  'yargs/helpers';`. `hideBin` strips the node + script-path argv prefix
  the same way yargs does internally, so the early check sees exactly
  what yargs would have parsed.
- `packages/cli/src/gemini.tsx:268-275` — new early-exit block runs
  *before* `startupProfiler.start('cli_startup')` and (more importantly)
  before `patchStdio`. Uses `writeToStdout((await getVersion()) + '\n')`
  followed by `process.exit(ExitCodes.SUCCESS)`. Newline matches
  conventional CLI output and what `$(gemini --version)` strips.

## Verdict
`merge-after-nits`

## Rationale
Right diagnosis (yargs prints via `process.stdout.write` which
`patchStdio` redirects), right fix location (before `patchStdio` runs),
right canonical version source (same `getVersion` the rest of the CLI
uses, so the two output paths can't diverge). Nightly verify-release
breakage is a real release-pipeline issue and this is the smallest
viable fix.

Nits worth raising:
1. **`-v` collision.** Many CLIs use `-v` for verbosity rather than
   version. If `gemini-cli` has any `-v` short-flag aliased to a verbose/
   verbosity option elsewhere, this early-exit will swallow it before
   yargs gets to disambiguate. Worth a quick `grep -r "alias.*'v'" packages/cli/src`
   to confirm `-v` is genuinely free.
2. **Naive contains check.** `earlyCliArgs.includes('--version')` will
   also fire if the user passes `--prompt --version` (a literal string
   value passed to a flag). For `gemini --prompt --version "explain this"`,
   the early-exit triggers and the prompt is never reached. Probably
   acceptable, but worth either documenting the limitation or doing a
   minimal positional check (e.g. only fire when `--version` is the
   first non-option arg).
3. **No regression test.** A vitest check that spawns `node gemini.js
   --version` and asserts the exit code + stdout would lock the
   verify-release contract in CI. Since the original failure was a
   release-pipeline regression, this is exactly the kind of thing that
   should have a test before re-shipping.

No banned strings; change is localized to one file.
