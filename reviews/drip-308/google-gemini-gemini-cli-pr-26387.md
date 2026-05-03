# google-gemini/gemini-cli #26387 — fix(core): implement system ripgrep fallback when bundled binary is missing

- PR: https://github.com/google-gemini/gemini-cli/pull/26387
- Author: chaitanyabisht
- Head SHA: `25518e8e8e395d545cc6d33fc5fb3d95a7ad8d2a`
- Updated: 2026-05-02T20:42:18Z

## Summary
Six-line addition to `getRipgrepPath` in `packages/core/src/tools/ripGrep.ts`: when the bundled vendor binary check fails, fall back to checking the system `PATH` for an `rg` executable using the existing `isBinaryAvailable` utility, and return the bare string `'rg'` so subsequent spawn calls resolve via PATH. Restores ripgrep performance for users on slim NPM tarballs that ship without architecture-specific binaries.

## Observations
- `packages/core/src/tools/ripGrep.ts` line ~65 (the new "3. Fallback to system PATH" block) returns the literal string `'rg'` rather than a resolved absolute path. That works for `child_process.spawn`, but anywhere downstream that does a path-equality comparison or treats the return value as a fully-qualified path may break. Consider using a `which`-style resolution to return the absolute path for consistency with the bundled-binary branch.
- The `isBinaryAvailable` import from `'../utils/binaryCheck.js'` is added at line ~41. Worth a quick look at whether `isBinaryAvailable` does an `access(X_OK)` check or a synchronous `execSync('which rg')` — the latter is significantly slower at process startup and `getRipgrepPath` may be on a hot path.
- No test in the diff. Easy unit test: stub `isBinaryAvailable` to return `true` after the bundled lookup misses, and assert `getRipgrepPath()` resolves to `'rg'`. Without this, the regression that motivated the PR ("Ripgrep is not available" warning when system rg is present) has no automated guard.
- Security note: returning `'rg'` to be later spawned without `shell: true` is fine, but flag it explicitly in a comment so a future change adding `shell: true` doesn't open a PATH-poisoning vector. The bundled-binary path returns a vetted absolute path; the fallback intentionally trusts user `PATH`.
- The fix is small, low-risk, and addresses a documented user-visible regression (`Fixes #26193`). After the lightweight tightening above it's ready.

## Verdict
`merge-after-nits`
