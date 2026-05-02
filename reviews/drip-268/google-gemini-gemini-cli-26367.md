# google-gemini/gemini-cli #26367 — fix(cli): print --version on real stdout before patchStdio

- **Head SHA:** `2525d9c18edd763e80eb663c8d47fca676c8a9dc`
- **Files:** `packages/cli/src/gemini.tsx` (+12/-0)
- **Verdict:** `merge-as-is`

## Rationale

Tight, well-justified fix. `patchStdio` in this CLI redirects `process.stdout.write` into the `coreEvents` bus so the React/Ink renderer can take ownership of terminal output. yargs' built-in `--version` handler writes via `process.stdout.write` *after* parse, which means once `patchStdio` runs, `gemini --version` emits nothing on the real stdout and breaks any release verification script that does `$(gemini --version)`. The fix at `gemini.tsx:268` checks `hideBin(process.argv)` for `--version` or `-V` *before* `patchStdio` is wired, calls the existing `getVersion()` helper exported from `@google/gemini-cli-core`, writes via `writeToStdout`, and exits 0 — which is exactly yargs' default `--version` semantics, just executed on the unpatched stdout.

The 12-line patch is purely additive in `main()` and runs before any other startup work (config load, profiler start, IPC listeners), so there's no ordering risk. `hideBin` and `getVersion` are already imported via `@google/gemini-cli-core`, so the bundle impact is zero. The early-exit means `cliStartupHandle = startupProfiler.start('cli_startup')` doesn't get an orphan `start` event paired with no `end` — also nice.

Two trivial follow-ups, neither merge-blocking: a comment-style preference for `// eslint-disable-next-line no-process-exit` if the repo has that lint, and a regression test under `packages/cli/src/__tests__` that asserts `await main()` exits 0 with the version string when `process.argv = ['node','gemini','--version']`. Merge as is; this unblocks `verify-release` immediately and the test can land separately.
