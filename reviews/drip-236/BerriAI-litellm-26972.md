# BerriAI/litellm#26972 — fix(ui): virtual keys filter silently resets after navigating away and back

- **PR**: https://github.com/BerriAI/litellm/pull/26972
- **Head SHA**: `f301297125b1bf53eab88a9829a9f884558d5b6a`
- **Size**: +206 / -264, 4 files
- **Verdict**: **merge-after-nits**

## Context

Virtual Keys page ran two parallel data paths for the same list:

1. `useKeys(page, size, { sortBy, sortOrder, expand })` — paginated full key list, no filter args.
2. `useFilterLogic({ keys, teams, organizations })` — filter UI state + a one-off `keyListCall` with filters writing to local `filteredKeys`, plus an effect on `[keys, filters]` that re-derived `result = [...keys]` and applied client-side `filter()` only for `Team ID` and `Organization ID`.

The bug: type `"foo"` into Key Alias → `debouncedSearch` fires server-filtered `keyListCall` → `filteredKeys = server result` ✓. Navigate away. `useKeys` re-runs (30s `staleTime` expiry, `storage` event, window focus, remount). New `keys` prop arrives → `[keys, filters]` effect re-runs → `result = [...keys]` (unfiltered) → no client-side filter branch for Key Alias/User ID → `setFilteredKeys(result)` overwrites the server-filtered list with the full page. Filter chips still show `"foo"` but the table lists everything.

## What's right

**Architectural fix: route filter values through the main `useKeys` query so React Query's cache key tracks them.**

Old surface (deleted): `useFilterLogic` owned a parallel `debouncedSearch → keyListCall → filteredKeys` path that was completely independent of React Query's cache. Any cache-driven refetch of `useKeys` would silently overwrite `filteredKeys`.

New surface at `VirtualKeysTable.tsx`:

```diff
  const { data: keys, ... } = useKeys(page, size, {
    sortBy, sortOrder, expand: "user",
+   teamID:           debouncedFilters["Team ID"]         || undefined,
+   organizationID:   debouncedFilters["Organization ID"] || undefined,
+   selectedKeyAlias: debouncedFilters["Key Alias"]       || undefined,
+   userID:           debouncedFilters["User ID"]         || undefined,
  });

- const { filters, filteredKeys, filteredTotalCount, ... } = useFilterLogic({
-   keys: keys?.keys || [], teams, organizations,
- });
+ const { filters, ... } = useFilterLogic({ teams, organizations });

- data: filteredKeys,
+ data: keys?.keys ?? [],

- const totalCount = filteredTotalCount ?? keys?.total_count ?? 0;
+ const totalCount = keys?.total_count ?? 0;
```

Same `(page, filter, sort)` tuple → same React Query cache entry. Any remount or refetch naturally pulls the filtered page from cache. The UI and the data can no longer drift apart because they're now reading from the same source-of-truth keyed by the same query parameters.

**`useFilterLogic` reduced to a pure state holder.** The `keys` prop, the `debouncedSearch` branch, the `[keys, filters]` effect, `filteredKeys`, and `filteredTotalCount` are all removed. What remains: `filters`, `handleFilterChange`, `handleFilterReset`, plus the existing `allTeams` / `allOrganizations` bootstrap. This is the correct narrower contract — `useFilterLogic` was reaching across two abstraction layers (filter UI state + filtered-data derivation), and the fix collapses it to just the first.

**300ms debounce sits between `filters` and the values fed into `useKeys`.** `FilterComponent` fires `onApplyFilters` on every keystroke of the text inputs (Key Alias, User ID), so without debouncing each character would trigger its own request. 300ms is the conventional sweet spot for type-then-pause UX.

**Pagination resets to page 1 when debounced filter values change.** Without this, a narrowed result set could leave the user stranded on an out-of-range page (filter "foo" reduces 100 keys to 3, but the user was on page 5). The reset is a small consumer-side correctness fix that the prior client-side architecture didn't need (it always re-derived from the same full `keys` prop).

**Test coverage:** `filter_logic.test.tsx` rewritten against the new contract with a regression test for "filter state survives re-renders" — the exact thing the bug exposed. `VirtualKeysTable.test.tsx` per-test mocks updated to feed `useKeys` data instead of `filteredKeys`, with two `filteredTotalCount` pagination cases collapsed to one server-filtered-total case + one assertion that filter values are forwarded into `useKeys`. The diff shows nine separate `mockUseKeys.mockReturnValue({...})` blocks added (e.g., at `:307-318`, `:355-366`, `:484-495`, `:534-545`, `:573-584`, `:618-629`, `:666-677`, `:716-727`, `:767-778`) — once per test that previously relied on `filteredKeys` to stage a single key into the table. PR body claims 30/30 passing.

## Risks / nits

- **`debouncedFilters["Key Alias"] || undefined`** at four lines: empty string `""` falls through `||` to `undefined` correctly, but `"0"` (User ID `"0"`) would also fall through. This is fine for the field semantics (no `"0"` user ID exists in practice) but worth a one-line comment that the `||`-vs-`??` choice is intentional, since a future User ID containing `"0"` as a prefix typed mid-stream would briefly show all keys.
- **`useKeys` cache key now includes 4 additional dimensions** — for power users with frequent filter toggling, this multiplies the cache-entry count. React Query's default `gcTime` (5min) bounds the memory growth, but worth confirming the hook's options don't pin all entries. Cosmetic.
- **`debouncedSearch`'s 300ms is internal to `VirtualKeysTable.tsx`** but the user-visible debounce on the existing `keyListCall` path was implicitly defined by `FilterComponent`'s emit cadence. Worth a one-line comment near the debounce hook explaining "300ms matches the type-pause threshold, do not lower without UX review" so a future contributor doesn't tune it down to 100ms and resurrect the per-keystroke fetch storm.
- **Removed `skipDebounce` hop on the sort handler** is a tangential cleanup (sort changes were going through a debounce-skip path that's no longer needed since sort is not a text input). Correct cleanup but worth one line in the PR body confirming it.
- **The `useFilterLogic` rewrite changes its return shape** — any other consumer of `useFilterLogic` (if it exists outside `VirtualKeysTable`) will break on `filteredKeys`/`filteredTotalCount` being gone. PR body lists 4 files changed, all under `VirtualKeysPage` and `key_team_helpers/`, so this likely is the only consumer. Worth a `rg useFilterLogic` confirmation.
- **`mockUseKeys.mockReturnValue` blocks repeat the same `KeysResponse` shape** 8 times across the test file. A helper `mockKeys(keys: Key[], totalCount = 1)` at the top of the file would deduplicate but doesn't block.

## Verdict

**merge-after-nits.** Architecturally correct fix routing filter values through React Query's cache key so the UI state and the displayed data can no longer drift across remount/refocus/staleTime-expiry boundaries. `useFilterLogic` collapsed to a pure state holder is the right narrower contract. Tests rewritten against the new contract with the regression case explicitly pinned. Want a one-line confirmation on `useFilterLogic` consumer count and a comment on the 300ms debounce intent before merge.
