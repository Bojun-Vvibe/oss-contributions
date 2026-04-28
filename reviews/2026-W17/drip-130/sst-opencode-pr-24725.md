# sst/opencode #24725 — fix(tui): sort session picker by full updated timestamp

- URL: https://github.com/sst/opencode/pull/24725
- Head SHA: `678357e2e1c997854a7fd981e6ad501ed8a263d6`
- Files: 1 (`packages/opencode/src/cli/cmd/tui/component/dialog-session-list.tsx`)
- Size: +1 / −5

## Summary

Replaces the two-tier `(updatedDay, createdMs)` comparator on the TUI session
picker with a single full-millisecond `b.time.updated - a.time.updated` sort.
The previous comparator at `dialog-session-list.tsx:113-118` truncated
`time.updated` to midnight via `setHours(0, 0, 0, 0)` before comparing, then
fell back to `time.created` as the intra-day tiebreaker — so a session that
was *touched* 30 seconds ago ended up beneath a same-day session that was
*created* later in the day, which is the exact opposite of what "sort by
recency of activity" means and the bug users actually filed.

## Specific references

- `packages/opencode/src/cli/cmd/tui/component/dialog-session-list.tsx:116`
  collapses to `.toSorted((a, b) => b.time.updated - a.time.updated)`. Numeric
  subtraction on epoch-ms is the right shape (no `Date` allocations per
  comparison, no `setHours` mutation cost — the prior comparator allocated
  two `Date` objects per pair).
- The "Today" / "Yesterday" / day-grouped headers downstream (typically
  rendered from `time.updated`) are unaffected because grouping is done by
  the renderer, not by the sort key — so removing the day-bucket primary key
  doesn't visually re-shuffle the day groupings.
- The `parentID === undefined` filter at `:115` is preserved verbatim, so the
  fix is scoped to ordering only.

## Risk

Trivially small. The only behavioral diff is "two same-day sessions where
`updated_a > updated_b` but `created_a < created_b`" — the new behavior
(activity-recency wins) is unambiguously correct. No test added, but the
function is one expression long; a regression would be visually obvious in
the picker on the next session-resume tick.

## Nits (non-blocking)

- A one-line unit-style assertion in `test/tui/dialog-session-list.test.tsx`
  pinning `[{updated:T+1, created:T-100}, {updated:T, created:T+1}]` →
  first-element-is-T+1 would cheaply protect the invariant against a future
  "let's re-add a created-time secondary key for stability" refactor.

## Verdict

`merge-as-is` — surgical correctness fix, smaller patch than the bug report,
no behavioral surprise.
