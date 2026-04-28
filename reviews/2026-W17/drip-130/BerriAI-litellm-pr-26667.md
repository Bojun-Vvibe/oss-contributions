# BerriAI/litellm #26667 — fix(ui): team members tab — O(1) lookup + client-side pagination & search

- URL: https://github.com/BerriAI/litellm/pull/26667
- Head SHA: `a3d496ad58742e18a0a06d856b80619e8869a5fc`
- Files: 3 (MemberTable component, TeamMemberTab component, test file)
- Size: +259 / −67

## Summary

Three layered fixes for a real UX freeze on large teams (PR description
reproduces with several hundred members; root-cause analysis names ~2000
members × 6 columns = 12,000 linear scans per render):

1. **O(n²) → O(n)**: replaces six per-row `Array.find()` calls over
   `team_memberships` with a single `useMemo`-built `Map<user_id, TeamMembership>`
   in `TeamMemberTab.tsx`.
2. **DOM bounding**: extends `MemberTable` with optional `withPagination`
   prop wiring antd `<Pagination>` controls and slicing the rendered rows
   to the current page.
3. **Filter UX**: adds an email/user-id search input plus an admin/non-admin
   role select, both threaded through a `useMemo`-derived `filteredMembers`
   computation.

## Specific references

- `TeamMemberTab.tsx:34-42` builds the lookup map with the right
  defensive `.filter((tm) => tm.user_id)` guard against null `user_id`
  (which would otherwise produce a `[null, tm]` map entry that collides
  with the next null-id row, silently shadowing data). The dependency
  array `[teamData.team_memberships]` is the right minimal scope —
  `searchText` and `roleFilter` don't affect membership data.
- `TeamMemberTab.tsx:44-58` `filteredMembers` correctly applies the
  role filter *before* the text filter and short-circuits with
  `if (!q) return true` after the role check, so `roleFilter`-only
  filtering doesn't pay the `.toLowerCase().includes()` cost on every
  member. The `role === "admin" || role === "org_admin"` discrimination
  matches what the rest of the dashboard treats as admin (good).
- `MemberTable.tsx:42-62` introduces `internalPage`/`controlledPage` dual
  shape — uncontrolled by default, controlled if `currentPage` is passed.
  This is the right component-API shape for migrating call-sites
  incrementally without forcing all of them to track page state.
- `MemberTable.tsx:50-56` correctly clamps `safePage = Math.min(page, totalPages)`,
  which is the load-bearing guard for the "user is on page 5, deletes
  enough rows to reduce total to 20, page 5 no longer exists" race —
  without this clamp, `pagedMembers` would be `[]` and the user would
  see an empty page with no indication why.
- `MemberTable.tsx:57-59` `countDisplay` correctly uses
  `totalMembers !== undefined && total !== totalMembers` to switch to
  "X of Y Members" when filtering is active — the `!== undefined` guard
  prevents the `0 of 0` confusion when the prop isn't passed at all.
- Test coverage at `TeamMemberTab.test.tsx:368-477` (truncated in this
  view but visible in the diff): five new tests covering email search,
  admin role filter, non-admin role filter, search-clear behavior, and
  filtered count display. All exercise the realistic user-event path
  (`userEvent.setup()` then `user.type`/`user.click`), not internal
  state mutation.

## Risk

Low. The `Map` lookup is strictly faster than the prior `Array.find()`
and produces identical values. The pagination is opt-in via
`withPagination={true}` so existing call-sites are unaffected. The
filter UI is purely additive (no API contract change).

## Nits (non-blocking)

- The `Map` is built from `team_memberships` keyed by `user_id`, but the
  table data source is `team_info.members_with_roles`. If a member exists
  in `members_with_roles` but not in `team_memberships` (likely for a
  newly added member before the join completes server-side), the lookup
  returns `undefined` and the spend/budget cells render `0`/`null`. This
  is the same behavior as before, but worth a comment naming the
  expectation so the asymmetry isn't surprising in 6 months.
- `defaultPageSize = 50` is the new default; `pageSizeOptions=["10","25","50","100"]`
  doesn't include 200 or 500 even though the bug report names "several
  hundred" members. A team-admin viewing 400 members will need 8 page
  flips at the default. Either bump the default to 100 for large teams
  or add 200/500 to the options.
- The diff has a stray blank line at `TeamMemberTab.tsx:23` (between the
  interface declaration and the `export default function`) — formatter
  drift, not a correctness issue.
- No test pins the `safePage` clamp behavior. A test that paginates to
  page 5, then mutates `members` to a length that forces `totalPages = 2`,
  and asserts the rendered rows are page-2 not empty would protect
  against a future "let's always trust `currentPage`" regression.
- The `<Input.Search>` component name was visible in the truncated diff
  but the import line at `MemberTable.tsx:3` only imports `Pagination`,
  not `Input` — so the search box must live in `TeamMemberTab.tsx`
  (which does import `Input` at `:8`). That's the right separation
  (search is team-tab-specific; pagination is generic), but worth
  confirming the search submit doesn't accidentally trigger a form
  submit on an enclosing form element.

## Verdict

`merge-after-nits` — substantial real perf fix with the right algorithmic
shape (Map for lookup, useMemo for filter, opt-in pagination prop), good
test coverage for the new filter paths, and the dual controlled/uncontrolled
pagination API is well-considered. The page-size ceiling and the
"member-without-membership" comment are easy follow-ups.
