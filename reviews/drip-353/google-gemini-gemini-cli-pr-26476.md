# google-gemini/gemini-cli #26476 — fix(ci): require nudge label before closing old PRs

- URL: https://github.com/google-gemini/gemini-cli/pull/26476
- Head SHA: `443d046069b97e6c2edb8496cf65e813d4351048`
- Author: gundermanc
- Size: +2 / -6 across 1 file

## Comments

1. `.github/scripts/gemini-lifecycle-manager.cjs:178-181` — Removing the `PR_CLOSE_DAYS = 14` constant and `prCloseThreshold` derivation is correct *for the new logic* but leaves the original "close after 14 days" intent un-coded. The new "Close" search now uses `updated:<${nudgeThreshold.toISOString()}` (i.e. "no activity for 7+ days *after* a nudge was sent") which is a different policy. Worth a comment block at the top of section 4 explaining the new policy in plain English: "PRs are nudged at 7d of inactivity; PRs that received a nudge and got no activity for another 7d are closed." The current diff is correct but cryptic.
2. `.github/scripts/gemini-lifecycle-manager.cjs:184-187` — The Nudge query change from `created:${prCloseThreshold.toISOString()}..${nudgeThreshold.toISOString()}` (a window) to `created:<${nudgeThreshold.toISOString()}` (everything older than 7d) is the **actual fix**. The previous window-based query had a bug: any PR older than `PR_CLOSE_DAYS` (14d) would fall *outside* the window and never get nudged, so PRs that aged past 14d without ever being nudged were never seen by the close pass either (because the close pass only fires after a nudge in the new world). The new "everything older than 7d that hasn't been nudged" is correctly inclusive.
3. `.github/scripts/gemini-lifecycle-manager.cjs:213` — Close query now requires `label:"status/pr-nudge-sent"` and `updated:<${nudgeThreshold.toISOString()}`. This is the real change: a PR can only be closed if (a) it was previously nudged, AND (b) there's been no update for 7 days since. This is much more humane than the previous "close at 14d regardless" policy — a contributor who pushes a commit after the nudge resets the close clock. Excellent UX change.
4. `.github/scripts/gemini-lifecycle-manager.cjs:213` — Slight concern: `updated:<${nudgeThreshold.toISOString()}` measures "no activity in the last 7 days," but the nudge itself is an *activity* (a comment + label). Confirm via GitHub API behavior that adding the `status/pr-nudge-sent` label and posting a nudge comment **bumps** `updated_at`, so the close clock genuinely starts from the nudge moment rather than from the pre-nudge last-activity time. If labeling doesn't bump `updated_at`, then a PR could theoretically be nudged at day 7 and closed at day 8 (because its underlying `updated_at` was already 7 days stale at nudge time). I believe both labeling and comments do bump `updated_at`, but worth a one-line code comment confirming that's why this works.
5. Variable cleanup: `prCloseThreshold` is gone, but verify nothing else in the file still references it. The diff window I see only shows two call sites; if there's a third use elsewhere in the file, this would now be `ReferenceError`.
6. Missing: a unit test or at least a CI dry-run note in the PR body. Lifecycle-manager scripts that auto-close contributor PRs are exactly the kind of code where a regression silently nukes valid contributions. A `--dry-run` mode that logs "would close PR #N" without acting would let maintainers verify the new policy on a real corpus before flipping it live.
7. PR title "fix(ci): require nudge label before closing old PRs" perfectly captures the substantive change. Good.

## Verdict

`merge-as-is`

## Reasoning

Tight, correct, contributor-friendly fix that turns an aggressive auto-close policy into a humane "warn then close if still no response" flow. The diff is +2/-6 with clear semantics. The clock-bump-on-label question (point 4) is worth confirming in a follow-up comment but doesn't block — worst case the policy still errs on the side of *not* closing, since labels do bump `updated_at` in GitHub's API. The dry-run-mode suggestion is a nice-to-have for next time.
