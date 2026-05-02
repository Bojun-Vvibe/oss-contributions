# google-gemini/gemini-cli #26367 — fix(cli): print --version on real stdout before patchStdio

- URL: https://github.com/google-gemini/gemini-cli/pull/26367
- Head SHA: `3f69fde5ac9f705b459fd4df354162353face484`
- Author: @Ritaldojr7
- Stats: small — single file, ~12 added lines

## Summary

`patchStdio` redirects `process.stdout.write` into the cli's coreEvents
bus. Yargs writes `--version` output via `process.stdout.write`, so a
plain `gemini --version` invocation emitted nothing on the real stdout
— catastrophic for `verify-release` style `$(gemini --version)` shell
captures. This PR detects `--version`/`-v` BEFORE `patchStdio` runs
and writes the version directly to the un-patched stdout.

## Specific feedback

- `packages/cli/src/gemini.tsx:268-274` — gating logic is the
  smallest possible correct fix: `hideBin(process.argv)` strips the
  node binary + script path, then a literal `.includes('--version') ||
  .includes('-v')` short-circuits before any side-effecting setup.
- Caveat with the `.includes(...)` shape: if a user's prompt or file
  argument legitimately contains the literal string `--version` (e.g.
  `gemini "release --version"`), this will mis-trigger. `hideBin`
  doesn't parse — it just strips argv[0] and argv[1]. Yargs' actual
  parser would correctly distinguish a positional from a flag. Worth
  hardening: either look only at `argv[2]` (the first real arg) or
  do a quick `parseArguments(early=true)` in version-only mode.
- `getVersion()` is now imported eagerly. Confirm it's a cheap import
  (no side effects at module-eval time) — if it pulls in heavy
  dependencies that the new fast-path was meant to avoid, the
  optimization is partially lost.
- `writeToStdout((await getVersion()) + '\n')` — uses an awaited
  function that previously was inside `main()`'s try/catch. The new
  call has no error handler. If `getVersion()` throws (e.g. corrupt
  package.json in a vendored install), `--version` would crash with
  an uncaught promise rejection rather than yargs' graceful fallback.
  Wrap in try/catch and fall through to the existing `main()` path on
  failure.
- `process.exit(ExitCodes.SUCCESS)` immediately after — bypasses every
  `process.on('exit')` handler registered later in `main()`. That's
  the *desired* behaviour (we want to be cheap and side-effect-free)
  but worth a comment so a future contributor doesn't try to add
  cleanup hooks they expect to run.
- No tests. A simple `cli` smoke test (`node dist/index.js --version`
  and assert non-empty stdout) would lock this in. Without one, the
  next refactor of `patchStdio` could re-break the exact symptom this
  PR fixes.

## Verdict

`merge-after-nits` — correct diagnosis and minimal-blast-radius fix
for a release-tooling foot-gun. Add the try/catch + a smoke test, and
consider hardening the arg detection against positional false-positives.
