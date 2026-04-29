# BerriAI/litellm PR #26735 — fix(ui): stop page.tsx prefetch from overwriting teams table pagination

- Repo: BerriAI/litellm
- PR: https://github.com/BerriAI/litellm/pull/26735
- Head SHA: `95a10795a3479ceb2883b4e1909470039e21ffea`
- Author: Bytechoreographer
- Size: +108 / −184, 3 files

## Context

The dashboard's top-level `page.tsx` was prefetching the *full* teams list
into local component state (`teams` / `setTeams`) and threading those props
down into `OldTeams`. `OldTeams` already calls `useTeams()` internally, which
issues a paginated `teamListCall(page=1, page_size=10)` and owns its own
table-state. The two stores fought: every time `page.tsx` re-prefetched
(navigation, focus events, hot reload), it would overwrite whatever page
the user had paged to in the table with the unpaginated full set, snapping
the table back to "page 1, all teams" and breaking the pager UI.

The PR fixes this by deleting both the `teams` prop and the `setTeams` prop
from `OldTeams`'s callsite at `page.tsx:535-537` and from the component's
public API, leaving `useTeams()` as the single source of truth for the
table.

## What the diff actually does

1. `ui/litellm-dashboard/src/app/page.tsx:533-537` — drops `teams={teams}`
   and `setTeams={setTeams}` from the `<OldTeams ... />` jsx. (Diff
   excerpt confirms only these two prop assignments are removed; the
   surrounding `accessToken / userID / userRole` props remain.)
2. The test file at `ui/litellm-dashboard/src/components/OldTeams.test.tsx`
   gets restructured: every test that previously passed `teams={[...]}` /
   `setTeams={vi.fn()}` directly into `<OldTeams ... />` now instead
   stubs `teamListCall` (the function `useTeams` calls under the hood) via
   a new `mockTeamsResponse(teams)` helper at lines 130-138. The helper
   shapes the response as `{teams, total, page: 1, page_size: 10,
   total_pages: ...}` to match the API contract `useTeams` consumes.
3. The empty-state tests at lines 422-432 just drop the props (no mock
   setup needed because `useTeams()` defaults to an empty list when the
   call hasn't resolved).

The structural pattern is consistent: **anywhere a test wanted to assert
"OldTeams renders with these teams", it now stubs the API instead of the
prop**. That's the correct direction — it pins the same flow production
takes — but it also means tests now depend on `vi.mocked(teamListCall)`
firing in the right order (`mockResolvedValueOnce` is per-call).

## Risks and gaps

1. **`mockResolvedValueOnce` on a `useQuery`-wrapped call is fragile to
   refetch**. If `useTeams()` does any reactive refetch on prop change /
   focus / interval (typical for `react-query` setups), the second invocation
   in the test will fall through to the next mock or to the default
   `vi.fn()` return (probably `undefined`), at which point `useTeams`
   could throw on the destructure. A `mockResolvedValue` (no `Once`) for
   tests that don't care about exact call counts would be safer; reserve
   `mockResolvedValueOnce` for the tests that actually assert "first call
   returned X, second call returned Y".

2. **No assertion that `OldTeams` no longer references `teams` or
   `setTeams` props in its TypeScript surface**. The test file deletes
   prop assignments, but if the `OldTeamsProps` interface still declares
   `teams?: ...` / `setTeams?: ...` as optional, downstream consumers
   (other dashboard pages, custom forks) will keep passing them and the
   props will silently no-op. Either delete the fields from the prop
   interface (with TypeScript catching dead callers) or keep them and
   `console.warn` once on receipt to flag drift.

3. **`page.tsx` still computes `teams` / `setTeams` somewhere**. The diff
   only shows the prop removal at the JSX site; the corresponding
   `useState<...>([])` and the network call that populated it are
   presumably still in `page.tsx` if no other consumer needs them. If
   nothing else in `page.tsx` reads `teams`, those state hooks and the
   prefetch network call are now dead weight that will keep firing on
   every render — a follow-up cleanup pass (or the same PR widening to
   delete the dead state + fetch) would close the loop.

4. **The empty-state tests drop the `teams` prop without setting up a
   mock**. That's a bet that `useTeams()` returns a sane default before
   resolving — typically `data: undefined` or `data: { teams: [] }`
   depending on the `useQuery` initial-data setting. If the component
   distinguishes "loading" from "empty" (it should, for UX), the
   "empty state" test may now actually be a "loading state" test in
   disguise. Worth either an explicit `mockTeamsResponse([])` setup or a
   `await waitFor(...)` that pins the assertion to the post-resolve
   render.

5. **No regression test for the actual reported bug** — i.e. "user
   navigates to page 3, an external prefetch fires, table stays on page
   3". Adding a test that mounts `<OldTeams>`, advances to a different
   page via the table's pager, fires a `teamListCall` mock (or a manual
   `useTeams.refetch()`), and asserts the table stays on the user's
   chosen page would lock the original bug behavior in.

## Suggestions

- Prefer `mockResolvedValue` over `mockResolvedValueOnce` for tests that
  don't need to assert call ordering, to harden against `useQuery`
  refetch.
- Delete `teams?: ...` / `setTeams?: ...` from `OldTeamsProps` so dead
  callers fail at typecheck instead of at runtime.
- Audit `page.tsx` for the now-dead `useState`/fetch that populated
  `teams`, and clean up in the same PR (or file a follow-up issue).
- Convert at least one empty-state test to use `mockTeamsResponse([])`
  with `await waitFor(...)` to pin the post-resolve render.
- Add a regression test for the original bug: page-to-N, fire refetch,
  assert pager stays on N.

## Verdict

**merge-after-nits** — the structural fix (drop the dual-source-of-truth
prop drilling, let `useTeams` own the data) is correct, and the test
migration is the right shape. Nits are about test robustness against
refetch and closing the loop on the now-dead `page.tsx` state, neither
blocks merge.
