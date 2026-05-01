# google-gemini/gemini-cli#26325 — Fix defense techniques 25099

- **PR**: https://github.com/google-gemini/gemini-cli/pull/26325
- **Head SHA**: `a7128198b27d7efb05333fd23437aa4e895e3449`
- **Verdict**: `request-changes`

## What it does

Per the PR title ("Fix defense techniques 25099") and the most recent
commit message ("Integrated decoherence check and fixed indentation for
PR #25190"), this *intends* to be a small fix to a `resolvePath`
ENAMETOOLONG normalization issue. In practice the diff is +19230 / -4206
across 200+ files, with the vast majority of changes being upstream-main
commits that the author appears to have rebased through, plus one tiny
behavioral change (`fix: skip normalization for long strings to avoid
ENAMETOOLONG crash` per commit `ff87114895e58f8bd1b31957a87aa72dcc287906`).

The actual diff body is too large for `gh pr diff` to return
(`HTTP 406: Sorry, the diff exceeded the maximum number of lines (20000)`).

## What's load-bearing

- **The branch shape is wrong.** The commit log shows ~70+ upstream-main
  commits (all with `committedDate` of 2026-04-23T18:21:42Z, the same
  rebase timestamp) followed by one author commit on 2026-05-01. This
  means the PR's `main` branch has fallen far behind upstream and the
  author rebased it forward, but in a way that surfaces every upstream
  change as part of *this* PR's diff to reviewers.
- **The single load-bearing change** appears to be the addition of an
  ENAMETOOLONG-aware path-normalization guard (per commit
  `3f7ac8c38f398695f8c04c2bd9c5e8204611c9dc` "Improve path normalization
  with error handling"). I cannot verify the actual line-level shape
  because the diff is unreviewable at this size.
- **Unrelated config/SVG snapshot churn dominates the file list:**
  changelogs (`docs/changelogs/latest.md` +250/-404,
  `docs/changelogs/preview.md` +236/-247), entire CI workflow deletions
  (`.github/workflows/ci.yml` +0/-499), `package-lock.json`
  (+458/-521), 30+ snapshot SVG files, etc. None of these are the
  author's intent; they're carry-along from the upstream rebase.

## Concerns (blocking)

1. **The PR is unreviewable in its current shape.** A 19,230-line diff
   for what the author describes as a one-function fix means the
   reviewer cannot distinguish "intentional change" from "carry-along
   from rebase." This is the single biggest reason to request changes.
2. **PR title/body don't match the diff scope.** The body is a blank
   template (no `Why`, no `What changed`, no `How to validate`); the
   title references issue numbers (#25099, #25190) without explanation
   of the relationship.
3. **Upstream-author-name mixing in commit history.** The
   `committedDate` for 70+ commits is 2026-04-23T18:21:42Z, which is
   the rebase moment, but `authoredDate` and `authors` are preserved
   from the original upstream PRs. This means the merge commit will
   show those original authors as co-authors of *this* PR, which is a
   process problem.
4. **No visible test coverage for the actual fix.** Without being able
   to read the diff, I cannot find tests for the ENAMETOOLONG guard.
   The author's commit message ("docs: remove redundant comments")
   doesn't suggest test coverage was added.

## Recommended path

1. Author should reset the branch against current upstream `main` so
   only the actual intended commits appear in the PR diff.
2. Re-fill the PR body with the standard template (Why, What changed,
   How to validate).
3. Add at least one regression test for the ENAMETOOLONG path-length
   case the original commit was guarding against.
4. Either reference issue #25099 with a one-line summary of the
   relationship, or remove it from the title.

## Verdict

`request-changes` — not because the underlying fix is wrong (I cannot
tell), but because the PR cannot be reviewed at 19,230 lines and the
template body provides no scope. Please rebase/squash the upstream
churn out and re-submit; happy to review the focused change.
