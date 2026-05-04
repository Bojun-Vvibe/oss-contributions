# QwenLM/qwen-code #3820 — fix(core): unescape shell-escaped file paths in Edit, WriteFile, and ReadFile tools

- SHA: `98678225516b813c29774bfe904040efa6b68c92`
- State: OPEN, +175/-7 across 7 files

## Summary

When the LLM produces shell-escaped paths (`my\ file.txt`, `project\ \(v2\).txt`) — typically because at-completion surfaces shell-quoted strings — the file tools fail with "file not found." PR routes `params.file_path` through `unescapePath()` inside `validateToolParamValues` of `EditTool`, `ReadFileTool`, `WriteFileTool`, and inside `CoreToolScheduler`'s conditional-rules matcher. New tests cover escaped spaces, multiple escape sequences, and literal-backslash preservation.

## Notes

- `packages/core/src/tools/edit.ts:592-594`, `read-file.ts:386-389`, `write-file.ts:407-409` — same three-line pattern in each: comment + `unescapePath(params.file_path)` + assign back. Mutating `params.file_path` in place is convenient for downstream code that re-reads `params`, but it's a side-effect inside a function named `validate*`. Two micro-concerns: (1) callers that retain the original `params` reference will see it mutated; (2) `validateToolParamValues` is documented elsewhere as returning a validation string only. A short jsdoc note "also normalizes file_path" would prevent surprises.
- **The PR diff does not contain the implementation of `unescapePath` itself.** Every changed file imports it from `../utils/paths.js`, but `packages/core/src/utils/paths.ts` is not in the file list. Either it already exists upstream (in which case its semantics are critical and should be linked from the PR body) or the implementing commit is missing from this branch — must reconcile before review can complete. Assuming it exists: the tests at `edit.test.ts:236-272` assert `\\\\` (literal double-backslash) is preserved while `\\ `, `\\(`, `\\)`, `\\&` are stripped. Those tests effectively pin the contract; reviewers should re-read `unescapePath` against them.
- `packages/core/src/core/coreToolScheduler.ts:1698` — `matchAndConsume(unescapePath(filePath))` correctly normalizes before conditional-rule lookup so a rule registered against the real path matches the LLM's escaped variant. But `filePath` itself is not reassigned, so other downstream consumers of the same `filePath` variable in this scope may still see the escaped form — verify or factor `const normalized = unescapePath(filePath)` and use `normalized` consistently.
- `packages/core/src/tools/edit.test.ts:266-272` — the literal-backslash preservation test uses `path.join(rootDir, 'path\\\\with\\\\slashes.txt')` which on Windows will produce native separators, partially defeating the test intent. Worth adding a `process.platform !== 'win32'` skip or constructing the literal with `String.raw\`...\`` to be unambiguous.
- `packages/core/src/tools/edit.test.ts:949-989`, `read-file.test.ts:283-300`, `write-file.test.ts:474-499` — solid end-to-end tests that create a real file with spaces, pass the escaped path, and assert the read/write/edit succeeds. Good coverage.
- Testing matrix in PR body shows only macOS unit tests verified. The escape semantics differ between POSIX shells and Windows cmd/PowerShell (where `\` *is* the path separator). Validation on Windows is critical before merge — `unescapePath` could regress legitimate Windows paths if it's not platform-aware.

## Verdict

`needs-discussion` — fix is well-motivated and tests are thorough, but two issues need resolution before merge: (1) the `unescapePath` implementation is not in this diff and its contract must be confirmed against the assertions in the tests; (2) Windows behavior is unverified and the escape-semantic mismatch is a real regression risk. Once those are addressed and the in-place `params.file_path` mutation is documented, this is a clear `merge-after-nits`.
