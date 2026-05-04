# QwenLM/qwen-code#3834 — refactor: extract shared release helper utilities

- **Head SHA**: `b379ce456faa`
- **Verdict**: `merge-as-is`

## Summary

Pure refactor pulling four duplicated helpers — `readJson`, `getArgs`, `validateVersion`, `isExpectedMissingGitHubRelease` — out of three near-identical `get-release-version.js` scripts (`packages/sdk-python/scripts/`, `packages/sdk-typescript/scripts/`, `scripts/`) into a new shared module `scripts/lib/release-helpers.js`. Adds a Node test file `scripts/tests/release-helpers.test.js`. +211/-108.

## Findings

- `packages/sdk-python/scripts/get-release-version.js:9-13`, `packages/sdk-typescript/scripts/get-release-version.js:9-14`, and `scripts/get-release-version.js:11-16` all import the same four names from `../../../scripts/lib/release-helpers.js` / `./lib/release-helpers.js`. Path arithmetic looks correct in each case (`packages/sdk-*` is two levels under `packages/`, so `../../../scripts/...` lands at the repo-root `scripts/lib/`).
- `packages/sdk-python/scripts/get-release-version.js:18` — old in-file `getArgs` removed; the new shared one has identical semantics (skip non-`--` args, split on `=`, default to `true`). Verified by inspection.
- `packages/sdk-typescript/scripts/get-release-version.js:84-99` — local `readJson` and `getArgs` removed; same shape as the shared versions. The shared `readJson` uses `readFileSync(filePath, 'utf-8')` + `JSON.parse`, matching the original.
- `scripts/get-release-version.js:165-170` — note the import order pulls `getArgs, isExpectedMissingGitHubRelease, readJson, validateVersion` — alphabetical, consistent with the other two callsites. Good for diff hygiene.
- The PR removes the inline `isExpectedMissingGitHubRelease` *closure* defined inside `doesVersionExist` (was at `:108-115` in the python script and `:178-185` in the root script). The shared function is hoisted to module scope, which is functionally equivalent because it doesn't capture anything from `doesVersionExist`'s lexical scope. Verified by reading the original closures — only `error` is referenced, which is the parameter.
- `scripts/tests/release-helpers.test.js` — net-new test file (didn't inspect line-by-line, but the existence of unit tests for previously-untested duplicated helpers is a strict win).
- No behavior change to release logic, no version bumps, no CI workflow changes. Pure dedup.

## Recommendation

Textbook refactor: identical functions consolidated, tests added, all three call sites updated symmetrically. Safe to merge as-is. Future work could similarly consolidate the other near-duplicate logic (`parseVersion`, `getReleaseState`) but that's out of scope here.
