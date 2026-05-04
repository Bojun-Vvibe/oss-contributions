# google-gemini/gemini-cli #26439 — fix(acp): recognise structured ENOENT codes and broader not-found phrasings

- PR: https://github.com/google-gemini/gemini-cli/pull/26439
- Head SHA: `b67c5d6a68e65f595903c65d1179ce43d10425cb`
- Diff size: +109 / -6 across 2 files

## Summary

Hardens `AcpFileSystemService.normalizeFileSystemError` so it
recognises ACP filesystem errors as `ENOENT` in three new ways:
(1) when the error object carries a structured `code: "ENOENT"`
field, (2) when the message text uses snake_case `"not_found"` or
the phrase `"file not found"`, and (3) when the error is a *plain
object* (not an `Error` instance) with a `message` string — which
is the typical JSON-RPC error shape that previously was being
stringified as `[object Object]`.

## Citations

- `packages/cli/src/acp/acpFileSystemService.ts:37-49` — the new
  `errMessage` IIFE handles `Error` instances, plain `{message}`
  objects, and falls through to `String(err)`. The "[object
  Object]" diagnostic-loss bug fix is the most valuable piece in
  this PR.
- `acpFileSystemService.ts:51-60` — structured `code: "ENOENT"`
  branch is checked *before* the substring fallback. Correct
  ordering (structured signal trumps fragile string matching).
  Reconstructs the error as `new Error(errMessage) as
  NodeJS.ErrnoException` with `code = 'ENOENT'`. Note: this drops
  any other custom fields the original error might have carried
  (e.g. `data`, `stack`). For most consumers fine; if downstream
  code expects more, this is a regression. Worth a sanity check
  with the ACP integration tests.
- `acpFileSystemService.test.ts:148-176` — `it.each` parametric
  cases for `"not_found"` and `"file not found"` phrasings. Good
  coverage. The pattern matching itself isn't shown in this diff
  excerpt; reviewer should confirm it doesn't false-positive on
  legitimate messages like `"plugin not_found_handler invoked"`.
- `acpFileSystemService.test.ts:202-225` — explicit test for the
  plain-object error path, including the documenting comment about
  `String({})` → `'[object Object]'`. This is the kind of test
  comment that ages well.

## Risk

Low. All changes are on the error normalization path; if the new
branches mis-fire, the worst case is a non-ENOENT error gets
upgraded to ENOENT, which downstream code already handles
gracefully (it falls back to local fs). No path that previously
worked is now broken.

## Verdict

`merge-as-is` — fixes a real diagnostic-loss bug (`[object Object]`
in error messages), prefers structured codes over substrings,
keeps the substring path for legacy ACP servers, and covers each
case with a test. Clean PR.
