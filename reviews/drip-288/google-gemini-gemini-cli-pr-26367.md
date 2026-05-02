# google-gemini/gemini-cli PR #26367 — fix(cli): print --version on real stdout before patchStdio

- Head SHA: `3f69fde5ac9f705b459fd4df354162353face484`
- URL: https://github.com/google-gemini/gemini-cli/pull/26367
- Size: +12 / -0, 1 file (`packages/cli/src/gemini.tsx`)
- Verdict: **merge-after-nits**

## What changes

`patchStdio` redirects `process.stdout.write` into the `coreEvents`
channel so the React/Ink UI can render output. yargs's built-in
`--version` handler writes via `process.stdout.write`, so when the
patch is in place `gemini --version` produced *no* output on the
real stdout — the nightly `verify-release` script captures
`$(gemini --version)` and was failing because the capture was empty.

The fix: handle `--version` / `-v` explicitly *before* `patchStdio` is
applied. Imports `getVersion` from `@google/gemini-cli-core` and
`hideBin` from `yargs/helpers`, peeks at `process.argv` early in
`main()`, and if the version flag is present, writes
`(await getVersion()) + '\n'` directly via `writeToStdout` and exits
with `ExitCodes.SUCCESS`. Closes #26359.

## What looks good

- The fix is minimal and correctly placed: it runs *before*
  `cliStartupHandle = startupProfiler.start('cli_startup')` (line
  ~275) which is the earliest point in `main()` that side-effects
  start. Pre-empting yargs entirely for this one flag is the right
  call rather than trying to teach `patchStdio` to whitelist
  `--version`.
- Using `hideBin(process.argv)` to derive the user-supplied args
  (line ~270) is consistent with how `parseArguments` consumes them
  later, so `node ... gemini --version` and `gemini --version` both
  hit the same code path.
- `writeToStdout((await getVersion()) + '\n')` matches the
  yargs-default trailing newline so the verify-release `$(...)`
  capture comes out byte-identical to the legacy un-patched
  behavior. That's the actual fix — previous shell consumers won't
  see a behavior change.
- `process.exit(ExitCodes.SUCCESS)` not `0` keeps the exit-code
  conventions consistent with the rest of the CLI.

## Nits

1. `earlyCliArgs.includes('--version') || earlyCliArgs.includes('-v')`
   (line ~272) is a substring-on-array check — it would also match
   `gemini some-prompt --version`, which is probably the right call
   (`--version` is non-positional in yargs and short-circuits the
   parse anyway). But it would *also* match `gemini "tell me about
   --version"` because the user prompt arg gets passed through.
   yargs would have ignored that case; the new check intercepts it
   and prints the version instead of running the prompt. Worth
   verifying with the existing `it("does not interpret quoted args
   as flags")` test pattern, or just gate this on
   `earlyCliArgs[0] === '--version'` for the strict-positional case.
2. `getVersion()` is `await`ed (line ~273). If `getVersion()`
   throws (network call? file read failure?) the user gets an
   unhandled promise rejection from `main()` instead of yargs's
   built-in fallback. Wrap in `try/catch` and fall through to the
   normal yargs path on failure so a transient `getVersion` bug
   doesn't completely break `--version`.
3. The PR body has a backslash-escaping artifact: `\patchStdio\
   redirects \process.stdout.write\ to coreEvents` — looks like
   intended `` `patchStdio` `` markdown that got mangled. Cosmetic,
   but worth fixing before merge so the commit message renders
   cleanly.
4. Tests: there's no test added for the new pre-patch path. A
   `vitest` for `main()` is heavy, but a small unit covering "if
   argv contains `--version`, getVersion is called and writeToStdout
   is called with `${version}\n`" would prevent regression. The
   verify-release pipeline catches the integration symptom but
   that's a long feedback loop.

## Risk

Low. The change is a pre-`patchStdio` early-exit — if the new branch
doesn't fire, behavior is identical to before. The only new failure
mode is the case where `getVersion()` throws, which is bounded by
nit #2 above. Worth a one-line CHANGELOG note since `--version`
output channel changed (even though the bytes are the same).
