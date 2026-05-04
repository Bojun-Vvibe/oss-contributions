# BerriAI/litellm #26972 — fix(ui): virtual keys filter silently resets after navigating away and back

- SHA: `f301297125b1bf53eab88a9829a9f884558d5b6a`
- State: OPEN, +206/-264 across 4 files
- Files: `ui/litellm-dashboard/src/components/VirtualKeysPage/VirtualKeysTable.tsx` (+33/-22), `…/key_team_helpers/filter_logic.tsx` (+18/-114), `…/VirtualKeysTable.test.tsx` (+121/-35), `…/filter_logic.test.tsx` (+34/-93)

## Summary

Fixes a UI desync on the Virtual Keys page where filter chips kept their values after navigating away and back, but the table reverted to the unfiltered list. Root cause: two parallel data paths (`useKeys` returning the unfiltered page + `useFilterLogic` running its own debounced server filter into a separate `filteredKeys` state). Fix collapses them into one — filter values are now part of the React Query cache key on `useKeys`, so any remount/refetch deterministically returns the filtered page. `useFilterLogic` is reduced to a pure state holder.

## Notes

- The diagnosis in the PR body is correct and well-articulated. The `useEffect([keys, filters])` only had Team/Org client-side branches, so when fresh `keys` arrived for Key Alias / User ID filters it overwrote the server-filtered list.
- `VirtualKeysTable.tsx` — debounced filters (300ms) are passed into `useKeys` options. Right call given `FilterComponent` fires `onApplyFilters` per keystroke.
- Pagination reset to page 1 when debounced filters change is mentioned in the PR body — please confirm the test suite covers "apply filter while on page 5 → goes to page 1" (couldn't see that case in the visible test diff).
- `filter_logic.tsx` — net -96 lines. Removing the `debouncedSearch` branch and the `[keys, filters]` effect eliminates the dual-source-of-truth bug entirely. This is the right shape.
- Test diff: every per-test `mockUseFilterLogic` block now drops `filteredKeys` / `filteredTotalCount` and adds an explicit `mockUseKeys.mockReturnValue(...)` returning `KeysResponse`. Mechanical but correct — keeps each test fully isolated.
- `VirtualKeysTable.test.tsx:206-208` — top-level `beforeEach` mock no longer sets `filteredKeys`; per-test `mockUseKeys` overrides drive the table data. Good pattern.
- The PR body claims a regression test was added — `filter_logic.test.tsx` rewrite is referenced. Worth one comment confirming "filter state survives re-renders" assertion is explicit (not just implied by the structure).
- Greptile confidence ≥ 4/5 was a stated pre-submission gate — assume that's in the PR comments.

## Verdict

`merge-after-nits` — solid fix with the right architectural collapse. Confirm: (1) explicit test for "filter applied → navigate away → return → table still filtered", (2) explicit test for pagination reset on filter change. If both are present in the test diff (the visible window cuts off at line ~870), this is `merge-as-is`.
