# sst/opencode PR #24913 — fix(edit): prevent empty oldString from overwriting existing files

- Repo: sst/opencode
- PR: https://github.com/sst/opencode/pull/24913
- Head SHA: `1d39b2d613c5c29ef1bfe4843cfe2ec2167640f5`
- Author: jeevan6996
- Size: +18 / −14, 2 files
- Closes: #23517

## Context

Per the existing contract of the Edit tool, `oldString: ""` was overloaded to
mean two different things depending on whether the target file existed: for a
non-existent path it meant "create the file with `newString`", and for an
existing path it silently fell through to a full-file replace. That overload
is the bug: a model that reaches for Edit to *append* a line, and that emits
`oldString: ""` thinking it's a no-op anchor, will instead clobber the entire
file with `newString`. The PR removes the second meaning at
`packages/opencode/src/tool/edit.ts:88-95` by throwing
`"oldString cannot be empty for existing files. Provide exact oldString
context for in-place edits."` once `existed === true`. New-file creation via
empty `oldString` is preserved.

## What the diff actually does

The production change is three lines: an early `throw new Error(...)` guarded
by `existed`, placed before the BOM-preserving read path. The test side flips
the previous behavioral assertion — what used to be
`preserves BOM when oldString is empty on existing files` (asserting the
clobber + BOM survival) is replaced at
`packages/opencode/test/tool/edit.test.ts:106-127` with
`throws when oldString is empty on existing files`, asserting the new
`rejects.toThrow("oldString cannot be empty for existing files")` plus that
the original `using System;\n` content is left intact (with BOM still
present). The deletions in the test are the old "expect the diff to contain
`-using System;` / `+using Up;`" assertions, which are correctly removed
because that path is now blocked.

## Risks and gaps

The fix is surgical and correct in isolation, but the surface area worries me
a little:

1. **No explicit "create" test alongside the new "throw" test.** The PR
   removes the only assertion that exercised the existing-file empty-oldString
   path and replaces it with a throw assertion, but I don't see a new positive
   assertion at the same level pinning that *non-existent* paths still flow
   through and create the file. There is presumably such a test elsewhere in
   the file, but for a behavioral fork this narrow it would be cheap to add a
   `creates new file when oldString is empty` test in the same describe block
   so the two halves of the fork are visible side by side and don't drift.

2. **Error class is a bare `Error`.** Other Edit failure modes in this file
   throw structured errors that carry tool-failure metadata (the `Tool.define`
   contract typically distinguishes user-fixable errors from defects). A
   plain `throw new Error(...)` may surface as a generic defect to the model
   rather than a recoverable, retry-with-different-args signal. Worth
   matching the surrounding error idiom — e.g. whatever
   `Tool.UserError` / `Tool.InvalidArgs` analogue this codebase uses — so the
   model retries with a real `oldString` instead of giving up.

3. **No prompt-side update.** The Edit tool's description (`edit.txt` /
   `DESCRIPTION`) almost certainly still documents `oldString: ""` as
   meaningful for existing files, or at minimum doesn't *forbid* it. If the
   description isn't updated to say "for existing files, `oldString` MUST be
   non-empty and must match exactly", capable models will keep emitting the
   bad shape and learn the rule by observation. A two-line description tweak
   in the same PR makes this fix self-consistent.

4. **Backwards-compat for any tool that legitimately wanted clobber.** If
   anyone *was* relying on `oldString: ""` + existing file as a "replace
   whole file" shortcut (e.g. a prompt template that regenerates a config
   file from scratch), this is now a hard break. The throw is the right
   answer — that pattern should use a Write tool, not Edit — but the PR body
   is silent on it. Worth a one-line CHANGELOG mention so the migration is
   discoverable.

## Suggestions

- Add the symmetric positive test (`creates file when oldString is empty and
  file does not exist`) in the same describe block so both halves of the fork
  are pinned in one place.
- Throw via the project's structured tool-error class so the model treats
  this as fixable input rather than a tool defect.
- Update `edit.txt` / Edit tool description to document the new requirement
  so models stop emitting the bad shape.
- One-line CHANGELOG / release note covering the regressed-to-error case so
  any upstream prompt template relying on the clobber surface migrates to
  Write.

## Verdict

**merge-after-nits** — the production change is correct and the test rewrite
captures the new contract. The nits (structured error class, prompt update,
symmetric create-test) are all small and improve robustness without
expanding scope.
