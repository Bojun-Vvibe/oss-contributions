# google-gemini/gemini-cli PR #26311 — # PR: Fix Lint, Stale Logic & Policy Conflict

- URL: https://github.com/google-gemini/gemini-cli/pull/26311
- Head SHA: `e9efec3a5b9db5ebf8d3f4f75f92380787d0ded0`
- Author: gemini-cli-robot
- Repo: google-gemini/gemini-cli

## Summary

Bot-authored rewrite of `gemini-scheduled-stale-issue-closer.yml` and
adjustments to `stale.yml` plus `tools/.../backlog_age.ts`. Replaces the
single-pass stale-and-close flow with a two-phase model: (1) un-stale or
close already-stale issues based on human activity since labeling, (2)
mark new candidates as stale.

## Specific notes

- `.github/workflows/gemini-scheduled-stale-issue-closer.yml:48-56` —
  introduces explicit constants `SIXTY_DAYS_MS`, `ONE_EIGHTY_DAYS_MS`,
  `GRACE_PERIOD_MS = 14 days`. Cleaner than the previous inline date
  math. Good.
- Phase 1 (`yml:60-117`) iterates issues already labeled `stale`,
  finds the most recent labeling event for `stale`, and checks for
  human comments/events **after** that timestamp. Two concerns:
  - The label name changes from `'Stale'` (capital S, the previous
    `batchLabel`) to `'stale'` (lowercase) at line 47. This is a
    breaking change for any historical issue still carrying the old
    `Stale` label — they will be invisible to the new query at
    line 119 (`-label:${STALE_LABEL}`). A one-time migration step
    (or accepting both via OR) is needed before this lands.
  - `staleEvent` is found via `events.reverse().find(...)`, but
    `listEvents` paginates oldest→newest by default; reversing to
    "most recent labeling" is correct only if `paginate` returns
    every page concatenated in chronological order. Confirm.
- Phase 2 (`yml:119-156`) uses
  `is:issue is:open -label:stale updated:<sixtyDaysAgo` and a per-run
  cap of `markedCount >= 50`. The cap is reasonable for rate-limit
  hygiene. Note: the prior implementation explicitly skipped issues
  with `>= 5 reactions`; that protection is missing in the new code
  path — re-add or document the deliberate removal.
- The new code excludes labels matching `maintainer / security /
  pinned / roadmap` (line 134) but the old code also exempted
  `'help wanted'` and the literal `'🗓️ Public Roadmap'`. The
  emoji-prefixed roadmap label won't match `roadmap` substring on
  the lowercased name? It will, since lowercasing preserves the
  substring. `'help wanted'` is **not** in the new exemption list —
  this is a regression unless intentional.
- `stale.yml` and `backlog_age.ts` changes weren't read in detail;
  flag for the maintainer to confirm scope is intentional given the
  PR title says "Lint, Stale Logic & Policy Conflict".
- Bot-authored PR. Maintainer should manually verify all three
  workflow edits before merge.

verdict: request-changes
