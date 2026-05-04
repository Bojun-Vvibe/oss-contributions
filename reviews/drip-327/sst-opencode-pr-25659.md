# sst/opencode #25659 — fix(app): show all subagent sessions in sidebar with collapsible chevron

- SHA: `1114600ea3f6b69ee7ecf772cca1cde91963dfba`
- State: OPEN, +132/-32 across 3 files

## Summary

Replaces the previous "show only most recent child session" behavior with a proper expandable list under each parent: a new `childSessions` helper returns all direct, non-archived children sorted newest-first, and the sidebar renders them under a collapsible chevron. New unit tests cover sorting, archive exclusion, empty-children, and exclusion of grandchildren.

## Notes

- `packages/app/src/pages/layout/helpers.test.ts:225-262`: tests are clean and cover the four meaningful branches. The "newest first" assertion uses `time.updated` for ordering — verify the implementation matches (test expects `child2` (updated:3) > `child3` (updated:2) > `child1` (updated:1)). Good.
- `childSessions(..., 120_000)` — the third argument looks like a recency window in ms. The PR description should clarify whether sessions outside that window are filtered or just deprioritized; the test passes them all through, suggesting it's just for grouping.
- Sidebar rendering changes are not in the inspected diff slice, but the helper API contract is clean and isolated.
- No telemetry or persistence implications — purely view-layer.

## Verdict

`merge-after-nits` — clarify the meaning of the `120_000` window arg in either a JSDoc on `childSessions` or by giving it a named constant (e.g., `RECENT_WINDOW_MS`). Test names and structure are otherwise solid.