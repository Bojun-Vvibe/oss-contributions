# charmbracelet/crush PR #2779 — fix(tools/view): honor limit parameter on files exceeding MaxViewSize

- Repo: `charmbracelet/crush`
- PR: #2779
- Head SHA: `65b74cd5cdfec06892aa6dde5a44fca2d74fd44a`
- Author: pragneshbagary

## Summary
The `view` tool refused to read any file larger than `MaxViewSize`
(100 KB) even when an explicit `limit` was passed. The fix adds
`params.Limit <= 0` to the size guard so the rejection only fires when no
limit is provided. Closes #2756.

## Specific references
- `internal/agent/tools/view.go:165` — guard changed from
  `if !isSkillFile && fileInfo.Size() > MaxViewSize` to
  `if !isSkillFile && params.Limit <= 0 && fileInfo.Size() > MaxViewSize`.
  One-line conditional widening; surrounding error message and skill-file
  bypass are unchanged. Comment "Only block if no limit is set." matches
  the new condition.

## Verdict
`merge-after-nits`

## Rationale
Correct, minimal fix matching the bug report. The combination of
`!isSkillFile && params.Limit <= 0 && fileInfo.Size() > MaxViewSize`
is the right precedence: skill files bypass entirely (preserved),
caller-bounded reads bypass the size check (new), and unbounded reads of
giant files still error out (preserved). Manual repro in the PR body
covers the four behavior cells.

Nits worth raising before merge:
1. **No new test.** The PR explicitly says "No new test added" and that
   the existing `view_test.go` suite "passes unchanged." A regression
   test for "large file + explicit `limit` returns first N lines" is
   exactly what stops this from regressing in three months. Should be a
   ~15-line addition to `view_test.go` writing a >100KB temp file and
   asserting `Limit=20` succeeds. Worth requesting.
2. **No upper bound on `params.Limit`.** A caller that sets
   `Limit=1000000` on a 5GB file will now bypass the size guard entirely
   and the downstream `readTextFile` will read up to that many lines.
   This is probably fine in practice (line-bounded reads are bounded by
   line count, not bytes), but worth confirming `readTextFile` doesn't
   read the entire file into memory before slicing — if it does, this
   fix trades one DoS surface for another. A `params.Limit <= MaxLimit`
   secondary guard would be defensive.
3. The comment "Only block if no limit is set." is accurate but a bit
   terse. Two-liner explaining "callers asking for a bounded slice are
   trusted to know what they want, so the size guard only fires for
   unbounded full-file reads" would age better.

No banned strings; behavior change is contained to the view tool.
