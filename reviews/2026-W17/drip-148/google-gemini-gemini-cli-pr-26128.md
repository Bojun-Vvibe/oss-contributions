# google-gemini/gemini-cli #26128 — fix(ux): added error message for ENOTDIR

- PR: https://github.com/google-gemini/gemini-cli/pull/26128
- Head SHA: `2ceec35569df6a6c0131f1865ae1e1ad186644b5`
- Author: devr0306
- Files: `packages/core/src/utils/fsErrorMessages.ts` (+3/−0), `packages/core/src/utils/fsErrorMessages.test.ts` (+13/−0)

## Observations

- This is a textbook minimum-diff bug fix: the existing `errorMessageGenerators` table at `fsErrorMessages.ts` already maps a dozen errno codes (`EACCES`, `ENOENT`, `EEXIST`, `ECONNRESET`, `ETIMEDOUT`, etc.) to user-facing strings, but `ENOTDIR` was missing, so `getFsErrorMessage` was falling through to whatever generic default produced the unhelpful "ENOTDIR: not a directory" raw passthrough cited in #23350. The PR adds the missing entry at `:55-57`, following the exact same `(path) => string` shape as the surrounding entries. No new code paths, no new exports, no API surface change.
- The error message wording at `:56` correctly describes the *cause* of `ENOTDIR` rather than just translating the code: "Check if the path is correct and that all parent components are directories." `ENOTDIR` is overwhelmingly emitted when a path component that the kernel expected to be a directory turns out to be a regular file (e.g., `/some/file.txt/inner` — exactly the path used in the test cell at `test.ts:11`). The wording teaches the user what to look for, not just what failed. This is the right UX bar for filesystem error messages.
- The two-arm `(path ? ... : ...)` pattern at `:56` matches the existing convention in this file — most other entries also conditionally include the path. The fallback string when `path` is undefined ("Not a directory.") is short and grammatical. Both arms end with the same "Check if the path is correct..." suffix, so the user experience is consistent regardless of whether the upstream caller threaded the path through. Good consistency.
- Test coverage is appropriately scoped: two cells at `:155-167` exercise both arms (with-path: `/some/file.txt/inner` → `Not a directory: '/some/file.txt/inner'. Check ...`; without-path: `Not a directory. Check ...`). The exact-string `expected` assertion is correct — the wording is user-facing UI and a silent rewording would now require an intentional test update. The test cells slot into the existing `testCases` table-driven harness, so they pick up the same path-substitution and code-routing exercise the other entries get.

## Risks / nits

- The PR title is `fix(ux): added error message for ENOTDIR` but the body says "Closes #23350" without quoting what the original symptom looked like. Worth one-line context in the PR body so a reviewer can see what the user was previously seeing (presumably the raw `ENOTDIR: not a directory` line bleeding through). Doesn't block merge.
- The "Pre-Merge Checklist" in the PR body shows only `MacOS / npm run` validated — not Linux/Windows. For a pure error-string addition this is fine (the code path is OS-agnostic), but the checklist accidentally implies the PR was only tested on macOS. The change is OS-independent because Node's errno-to-name mapping is consistent across platforms.
- No test cell for the `path: undefined` *empty string* edge case — only `path` is omitted entirely, not `path: ''`. The current truthiness check `path ?` correctly treats empty string as falsy (so it would route to the no-path arm), but a future maintainer changing the predicate to `path !== undefined` would silently change behavior for empty strings. A defensive cell with `path: ''` would pin the truthiness contract.
- Adjacent code organization nit: the `errorMessageGenerators` table is alphabetically sorted by errno name *up to* the new entry, which lands at the bottom rather than between `ENOENT` and `ENOSPC` (or wherever it would alphabetize). Trivial — preserves diff cleanliness — but worth a comment noting the table is "by errno name" if it's meant to be sorted, or "by add-order" if not.
- The PR claims "Added/updated tests" — accurate, this is the rare case where the test additions are commensurate with the code addition (both arms exercised). No further test work needed.

## Verdict: `merge-as-is`

**Rationale:** Minimum-diff fix that slots into an existing well-tested table-driven errno-to-message harness, with two test cells exercising both arms (path-present and path-absent) and exact-string assertions that pin the user-facing wording. No surface changes, no risk of regression elsewhere, OS-independent. The wording itself ("Check if the path is correct and that all parent components are directories.") correctly describes the *cause* of `ENOTDIR` rather than just translating the code, which is the right UX bar.

Date: 2026-04-29
