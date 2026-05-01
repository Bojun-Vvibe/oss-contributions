# google-gemini/gemini-cli#26311 — Fix Lint, Stale Logic & Policy Conflict (scheduled stale-issue closer)

- **Author:** gemini-cli-robot
- **Head SHA:** `e9efec3a5b9db5ebf8d3f4f75f92380787d0ded0`
- **Base:** `main`
- **Size:** +213 / -84 across 3 files
- **Files changed:** `.github/workflows/gemini-scheduled-stale-issue-closer.yml` (dominant), plus 2 others

## Summary

Rewrites the scheduled stale-issue-closer workflow from a one-pass "search-and-close" model to a two-phase mark-then-close model with a 14-day grace period. Phase 1 walks issues already labeled `stale`, removes the label on detected human activity, and closes after the grace threshold; phase 2 marks new candidates (open issues with no `stale` label and `updated:<60d`) so they enter phase 1 on the next run.

## Specific code references

- `.github/workflows/gemini-scheduled-stale-issue-closer.yml:48-55`: explicit constants `STALE_LABEL = 'stale'`, `SIXTY_DAYS_MS`, `ONE_EIGHTY_DAYS_MS`, `GRACE_PERIOD_MS = 14 * 24 * 60 * 60 * 1000`. Replacing the prior in-line `threeMonthsAgo`/`tenDaysAgo` Date arithmetic with named ms constants makes the policy auditable in one place — load-bearing for change-management on a workflow that auto-closes user issues.
- `:62-69`: phase-1 candidate query is now `github.rest.issues.listForRepo` with `labels: STALE_LABEL, state: 'open'` rather than the prior search-API approach. Correct shift — `listForRepo` with a label filter is server-side filtered, eliminating the prior client-side label-walk loop.
- `:73-77`: events-based stale-event localization (`events.reverse().find(e => e.event === 'labeled' && e.label?.name === STALE_LABEL)`) — `reverse()` on the paginated events array correctly grabs the *most recent* labeling, important if the label was removed and re-added (un-stale → re-stale flow). The fallback `if (!staleEvent) { core.warning(...); continue; }` at `:77-79` is the right defensive arm.
- `:84-94`: human-activity detection — both `comments.data.some(c => c.user.type !== 'Bot')` (post-stale comments) and `events.some(e => e.actor.type !== 'Bot' && new Date(e.created_at) > staleDate && e.event !== 'labeled')` (post-stale events that aren't labelings). The `e.event !== 'labeled'` exclusion is load-bearing — without it, the stale-labeling event itself would count as "human activity" and the unstale logic would be a no-op.
- `:95-107`: un-stale path correctly only fires when human activity detected post-stale-event, and the `removeLabel` call is gated on `!dryRun`. The `continue` at `:107` is critical so the issue is not also closed in the same iteration after un-staling.
- `:110-132`: phase-1 close path gated on `staleDate < graceThreshold` (i.e., labeled stale 14+ days ago). Three GitHub API calls in sequence (createComment / update state / [implicit close lock or label add omitted]). The hardcoded close comment at `:121` (`'This issue has been closed due to 14 additional days of inactivity after being marked as stale...'`) is user-visible policy — worth reviewer sign-off on the wording.
- `:138-152`: phase-2 mark loop. The query `is:issue is:open -label:${STALE_LABEL} updated:<${sixtyDaysAgo}` excludes already-stale issues so the work is non-overlapping with phase 1. The `if (markedCount >= 50) break;` safety at `:148` caps marks per run — load-bearing throttle to prevent a misconfigured 60-day cutoff from spamming hundreds of issues in one run.
- `:151`: standard exemption labels `'maintainer'`, `'security'`, `'pinned'`, `'roadmap'` (substring matched lowercased). The substring-match approach is correctly more permissive than the prior `lowercaseLabels.includes('help wanted')` exact-match, but `pinned` substring will also match `pinned-issue` etc. — intended? Worth a comment at `:151` explicitly stating substring semantics.

## Reasoning

This is a real policy improvement. The prior single-pass logic would close an issue on its first old-update detection without giving the reporter a chance to respond, and used the search API in ways that miss labeled-then-unlabeled history. The two-phase mark-then-close pattern is the standard GitHub stale-bot model for good reason: it gives users a 14-day notification window via the `stale` label appearing on their issue, after which silence is consent.

Concerns:
- **180-day cutoff for `help wanted` issues is declared (`hundredEightyDaysAgo` at `:54`) but I don't see it consumed in the visible diff window.** The 60-day cutoff is used at `:140`; the 180-day path either lives later in the file (out of my diff window) or was dropped during refactoring. If dropped, the `hundredEightyDaysAgo` constant should be removed; if used, verify the `help-wanted`/`good-first-issue` exemption is correctly applied to the longer cutoff.
- **`per_page: 100` on `github.rest.issues.listForRepo` at `:67` returns at most 100 items.** No pagination wrapper around it (the prior code used `github.paginate(...)`). For repos with >100 stale issues, this will silently process only the first 100. Wrap in `github.paginate(github.rest.issues.listForRepo, ...)` for symmetry with the events listing at `:74` which does paginate.
- **Phase-2 search uses `github.rest.search.issuesAndPullRequests` at `:140`** with `per_page: 100` — same single-page issue. The `markedCount >= 50` cap at `:148` masks this in practice (50 marks/run × 1 page is fine), but the inconsistency vs phase 1 is worth a comment.
- **Race between phase-1 un-stale and phase-2 mark.** If phase 1 removes the `stale` label from an issue that *also* hasn't been updated in 60 days (the un-staling actor only commented, didn't push or react), phase 2 in the *same run* will re-mark it. Probably benign (the label removal in phase 1 → label addition in phase 2 produces two label events, and the next run's phase 1 will treat the new stale-labeling as fresh and skip closing for 14 days), but worth a one-line comment confirming the intent.

## Verdict

**merge-after-nits**
