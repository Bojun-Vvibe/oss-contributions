---
pr_number: 26667
repo: BerriAI/litellm
head_sha: a3d496ad58742e18a0a06d856b80619e8869a5fc
verdict: merge-after-nits
date: 2026-04-28
---

# BerriAI/litellm#26667 — fix(ui): team members tab — O(1) lookup + client-side pagination & search

**What changed.** +259/−67 across 3 files in the React UI dashboard. Three independent layers bundled in one PR:

1. **O(1) membership map** in `TeamMemberTab.tsx`: replaces six per-cell `teamData.team_memberships.find((tm) => tm.user_id === userId)` calls with a single `useMemo`-cached `Map<userId, membership>` rebuilt only when `teamData.team_memberships` changes. Every column then does `membershipsMap.get(userId)`. With 2k members × 6 columns the per-render cost drops from ~12 000 array scans to ~12 hash lookups (PR body's number).

2. **Client-side pagination** in `MemberTable.tsx`: new `withPagination` / `defaultPageSize` / `currentPage` / `onPageChange` / `totalMembers` props with internal `useState` for page+pageSize when uncontrolled (`page = controlledPage ?? internalPage`, same pattern for `setPage`). Renders antd `Pagination` in a top-right header row, slices `members.slice((safePage - 1) * pageSize, safePage * pageSize)` before passing to `<Table>`. `Math.min(page, totalPages)` clamp at `:62` correctly handles the "current page is now past end after filter narrowed the dataset" case.

3. **Search + role filter** in `TeamMemberTab.tsx` (visible in test cells `:371-477`): real-time email/user_id substring search + Admin/Non-admin role dropdown, with a React `key` prop pattern that resets `MemberTable`'s internal pagination state on filter change without prop drilling.

Test surface in `TeamMemberTab.test.tsx` adds 4 cells at `:371-477` covering email-search, role-Admin filter, role-Non-admin filter, and (presumably) pagination + edit-modal-receives-correct-membership.

**Why it matters.** PR body's measurement is plausible: every render of a 2k-member team paid 12k linear scans across 6 columns. React re-renders `TeamMemberTab` on every state change (modal open, hover, parent re-render), so this compounded into multi-second freezes for larger teams. The Map lookup is O(1) per cell.

**Concerns.**
1. **Three independent changes in one PR** (perf fix, pagination, search/filter) — any one of them would be a clean standalone diff. Reviewer needs to verify each independently. Recommend splitting in future, but salvageable here because the test cells map cleanly to the search/filter layer.
2. **`controlledPage ?? internalPage` + `onPageChange ?? setInternalPage`** at `:55-58` is correct for the controlled/uncontrolled hybrid pattern, but the failure mode is invisible: a caller passing only `currentPage` without `onPageChange` gets a one-way data binding where clicking the pagination silently doesn't update the page (because `setPage = setInternalPage` writes to internal state but `page = controlledPage` still reads external). React's standard idiom is to throw or warn on this inconsistency. Either tighten the prop types (`{ controlledPage: number; onPageChange: (p: number) => void } | { controlledPage?: undefined; onPageChange?: undefined }`) or `console.warn` when one is set without the other.
3. **`pageSize` is local state at `:51`** (`useState(defaultPageSize)`) — no controlled-pageSize path. If a parent wants to remember pageSize across remounts (e.g. user preference), they can't. Acceptable for this PR but a known limitation.
4. **`Math.min(page, totalPages)` clamp** is the right shape but doesn't *update* `page` to `safePage` — next render still has stale `internalPage`. If filter narrows data while user is on page 7 of a (now) 3-page list, the table renders correctly (page 3 worth of rows) but the pagination control still shows "page 7". A `useEffect` guarding `if (page > totalPages) setPage(totalPages)` would close this. Or rely on the React `key`-prop reset pattern the PR mentions for filter changes (which forces unmount → fresh `internalPage = 1`). Confirm both code paths are tested.
5. **`countDisplay` ternary at `:64-66`** correctly shows "X of Y Members" only when filter is active (`total !== totalMembers`), else just "X Members". Good UX detail; verify the test cells assert this.
6. **`loading` prop** is plumbed through to `<Table loading={loading}>` at `:160` but no test cell exercises it. Trivial enough to skip.
7. **`pageSizeOptions={["10", "25", "50", "100"]}` are strings** because antd's `Pagination` types require strings. Standard antd quirk, not a bug.
8. **No memoization of `pagedMembers`** — the slice runs on every render. For 50 rows this is negligible, but pair with `useMemo([members, safePage, pageSize])` for hygiene.

Real perf fix, sensible pagination shape, good test coverage. Ship after tightening the controlled/uncontrolled-page invariant (concern 2) and confirming filter-narrowing doesn't strand the user on a non-existent page (concern 4).
