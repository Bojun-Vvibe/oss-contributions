---
pr: BerriAI/litellm#26810
sha: 93ef4f4e9f8364fc54b24ba4d21cce57fb1ba193
verdict: needs-discussion
reviewed_at: 2026-04-30T00:00:00Z
---

# fix(usage): remove timezone date expansion causing extra bar in Daily Spend chart

URL: https://github.com/BerriAI/litellm/pull/26810
Files: `litellm/proxy/management_endpoints/common_daily_activity.py`

## Context

`_adjust_dates_for_timezone` at `common_daily_activity.py:385` was adding
±1 UTC day to the query window to "capture all records that fall within
the user's local date range." The function's premise was: DB rows are
stored with UTC `startTime`, so a user in PST asking for "Feb 4" needs
Feb 4 00:00 PST → Feb 5 07:59 UTC, which spans two UTC dates.

This PR rips out that expansion and returns the dates unchanged, on the
grounds that the daily-aggregate column is a `YYYY-MM-DD` string already
extracted from UTC `startTime`, and the frontend sends *local* YYYY-MM-DD
strings, so the expansion was producing an extra phantom day-bar in the
Daily Spend chart.

## What's right about this

- The user-visible bug is real: any non-UTC user saw an extra bar at the
  edge of their selected range. `#26797` (drip-184) already fixed the
  single-day variant of this; this PR is the multi-day generalization.
- The new docstring at lines 388-401 is honest about the new contract:
  "frontend sends local dates formatted with the local year/month/day, so
  the dates already match what is stored." That is the only consistent
  story if you accept the assumption.

## What's worrying

The fix only works if the assumption "frontend sends local Y-M-D, DB
stores UTC Y-M-D, and we silently treat them as the same" is exactly what
every caller does. But:

- The DB column is derived from `startTime` (UTC). A request that arrives
  at 23:59 PST on Feb 4 lands in the DB under UTC date Feb 5. A PST user
  asking for "Feb 4" will now miss that row entirely. The prior expansion
  caught it (at the cost of an extra phantom bar). The new code drops it.
- The function still accepts `timezone_offset_minutes` and silently
  ignores it (line 401: "Unused; kept for interface compatibility"). That
  is a footgun: any future reader assumes the offset *does* something.
  Either remove the parameter or assert on it.
- No test in this PR. `#26797` added a test for the single-day case;
  this multi-day rewrite removes the underlying logic and ships no
  regression test for the "PST user asking for Feb 4 should see all PST-
  Feb-4 spend" case — which is the original feature this function existed
  to provide.

## Verdict

`needs-discussion` — the chart-bar bug is real and worth fixing, but
"return dates unchanged" trades the cosmetic phantom-bar for a silent
data-loss bug at the day boundary for any non-UTC user. The right fix is
likely either (a) bucket spend by *local* date in the aggregator, not in
the query window, or (b) keep the expansion but de-duplicate the phantom
bar in the post-processing step. At minimum: drop the unused parameter
and add a test for the cross-midnight case.
