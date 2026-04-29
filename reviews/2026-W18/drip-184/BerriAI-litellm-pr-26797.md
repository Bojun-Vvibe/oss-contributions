---
pr: BerriAI/litellm#26797
sha: 93ef4f4e9f8364fc54b24ba4d21cce57fb1ba193
verdict: merge-after-nits
reviewed_at: 2026-04-30T00:00:00Z
---

# fix(usage): single-day date filter shows extra day for non-UTC users

URL: https://github.com/BerriAI/litellm/pull/26797
Files: `litellm/proxy/management_endpoints/common_daily_activity.py`,
`tests/test_litellm/proxy/management_endpoints/test_common_daily_activity.py`
(plus an unrelated workflow_management_endpoints rebase)
Diff: 1054+/27- (date-filter portion is small)

## Context

The Daily Spend chart in the Usage UI was showing N+1 days of bars when
a user picked a single day in the date picker — e.g., picking
`2026-04-23` for a PST user produced bars for both `2026-04-23` AND
`2026-04-24`. Root cause was `_adjust_dates_for_timezone` in
`litellm/proxy/management_endpoints/common_daily_activity.py:385` which
took the JS-style `Date.getTimezoneOffset()` value the frontend sends
(positive for west-of-UTC) and bumped `end + 1 day` for west-of-UTC
users or `start - 1 day` for east-of-UTC users, on the theory that
"local evening extends into next UTC day so we need to widen the UTC
query range." The theory is wrong for this schema: the database stores
the date string as the YYYY-MM-DD slice of the UTC `startTime` column,
and the *frontend already sends local-formatted YYYY-MM-DD strings*, so
the dates already match what's stored. The widening just dragged in an
adjacent day's worth of records.

## What's good

- The fix at `common_daily_activity.py:385-407` is the smallest correct
  change: collapse the function body to `return start_date, end_date`,
  rewrite the docstring to explain *why* it's a no-op now ("frontend
  sends local-formatted YYYY-MM-DD that already matches the UTC
  YYYY-MM-DD slice in storage"), and keep the `timezone_offset_minutes`
  argument in the signature with a "Unused; kept for interface
  compatibility" note. Keeping the argument is the right call — every
  call site already passes it, ripping it out would force a coordinated
  multi-file diff that distracts from the bug fix.
- The `from datetime import datetime` at `:1` correctly drops the now-
  unused `timedelta` import — small but easy-to-forget housekeeping
  that prevents lint regressions.
- The new docstring at `:386-403` explicitly documents the *bug class*
  ("expanding the range by ±1 day based on the timezone offset causes
  an adjacent day's data to appear as an extra bar in the Daily Spend
  chart") so the next person who looks at this function and thinks
  "wait, why isn't this doing anything?" doesn't reintroduce the bug.
- Test additions at
  `test_common_daily_activity.py:60-80` lock the contract:
  - `test_adjust_dates_for_timezone_single_day_pst` — PST user
    (offset=480), single day `2026-04-23`, must return
    `("2026-04-23", "2026-04-23")` unchanged.
  - `test_adjust_dates_for_timezone_single_day_ist` — IST user
    (offset=-330), same input, same unchanged output.
  - `test_adjust_dates_for_timezone_no_offset` — `None` offset, same
    unchanged output.
  These three are the right matrix for catching a future "well-meaning
  refactor" that re-adds the widening on the theory that the no-op
  function "looks broken."

## Nits

- The PR diff also contains the entire `workflow_management_endpoints.py`
  + schema additions + tests for `/v1/workflows/runs` (~1000 LOC) which
  is a different PR's content (#26793 already reviewed in drip-182,
  needs-discussion verdict). This rebase artifact makes the actual
  date-filter fix harder to spot in code review and means anything
  blocked on the workflows PR (tenant-scoping, race condition, error
  detail leak — see drip-182 review) silently blocks this trivial fix
  too. Strongly prefer rebasing onto a clean base and re-pushing as a
  pure ~30-line PR.
- The `timezone_offset_minutes` parameter, now permanently unused,
  warrants a `# noqa: ARG001` or pylint disable so static analyzers
  don't flag the dead arg, AND a deprecation comment naming the
  follow-up PR that will eventually drop it from the signature once
  callers are updated.
- No frontend test in this PR. The bug surfaced as "UI shows extra
  bar"; the regression-fence at the API layer prevents the UTC
  expansion bug, but a frontend snapshot or e2e test for "single-day
  selection on the Usage chart shows exactly one bar" would catch a
  symmetric bug introduced on the rendering side (e.g., `+1 day` added
  in the chart-axis builder).
- Worth checking via grep that no other endpoint or background job
  calls `_adjust_dates_for_timezone` expecting the old expansion
  behaviour — the function is now a documented no-op, but if anything
  was relying on the side effect of "give me a slightly-wider range
  for safety", it's now silently narrower. The `_build_where_conditions`
  call site at `:404` (caller) is the obvious one but a one-line audit
  would close the loop.

## Verdict reasoning

Correct, surgical fix for a real visible UI bug — the fix is
"acknowledge the function should never have done what it was doing"
which is the right diagnosis (the frontend-formatting + UTC-slicing
storage shape make timezone expansion meaningless). Tests lock the
no-op contract across PST/IST/None offsets. The blocking nit is the
unrelated `workflow_management_endpoints` rebase noise sharing the PR
— that previously-reviewed file has open security/correctness
questions in drip-182 that shouldn't gate a clean date-filter fix.
Land after rebasing onto clean main.
