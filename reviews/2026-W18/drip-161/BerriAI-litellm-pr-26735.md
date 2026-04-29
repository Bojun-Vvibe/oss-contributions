# BerriAI/litellm PR #26735 — fix(ui): stop page.tsx prefetch from overwriting teams table pagination

- Repo: `BerriAI/litellm`
- PR: https://github.com/BerriAI/litellm/pull/26735
- Head SHA: `95a10795a3479ceb2883b4e1909470039e21ffea`
- State: OPEN, +108/-184 across 3 files (target branch `litellm_internal_staging`)

## What it does

Fixes a bug where the dashboard's `?page=teams` view was silently ignoring server-side pagination once `>10` teams existed in the proxy DB. Root cause: `app/page.tsx` was prefetching teams via the legacy `teams`/`setTeams` props and passing them down to `OldTeams`, which then re-fetched on its own paginated lifecycle — the parent's prefetch state stomped on the child's paginated reads on every parent re-render.

Fix: stop passing `teams`/`setTeams` from `page.tsx` to `OldTeams` entirely (deletes both prop wires at `app/page.tsx:533-538`). `OldTeams` becomes the sole owner of its team-list state, fetching via the existing `teamListCall` + `useTeams` hook with proper `page`/`page_size`/`total_pages` plumbing.

## Specific reads

- `ui/litellm-dashboard/src/app/page.tsx:533-538` — the entire change in this file: just the two prop drops:
  ```tsx
  ) : page == "teams" ? (
    <OldTeams
-     teams={teams}
-     setTeams={setTeams}
      accessToken={accessToken}
      ...
  ```
  Surgical and correct. The `teams` and `setTeams` state in the parent presumably still exists (other consumers may read `teams` for badge counts etc.); confirm a follow-up grep for residual references that now point at stale data.
- `ui/litellm-dashboard/src/components/OldTeams.test.tsx` — the bulk of the +108. New `mockTeamsResponse(teams: any[])` helper (lines ~134-143) injects `teamListCall` returning `{teams, total, page:1, page_size:10, total_pages: ...}` shape per test, replacing the old prop-injection pattern. Updated tests at lines 360-395 ("should clear the delete modal when the cancel button is clicked"), 405+ ("empty state"), and the `teams is null` case all migrate consistently — no test left passing the dropped `teams=`/`setTeams=` props.
- `mockTeamsResponse` uses `mockResolvedValueOnce` (line 132) — once-per-call. A test that triggers two refetches (e.g., delete + reload) would silently fall through to the un-mocked default. Several tests touch delete flows; confirm those re-stub on each expected refetch or convert to `mockResolvedValue` for the steady-state cases.
- The `total_pages: teams.length > 0 ? 1 : 0` derivation in the helper is a test-only convenience. That's fine for unit tests, but a 25-team test case would need a separate helper that paginates correctly to exercise the actual bug class this PR is fixing — otherwise the regression test surface only proves "renders 1 page", not "paginates correctly across 3 pages without parent stomp". A 30/30 pass count was advertised; verify at least one of those 30 simulates `total_pages: 3` to lock in the bug-fix invariant.

## Risk

- Behavioral change for anything else in `page.tsx` that previously read the parent-level `teams` state assuming it was always-fresh. If a sidebar badge counts teams via `teams.length`, it now reads the prefetch snapshot only. Worth a grep on the file for residual `teams.` accesses.
- Target branch is `litellm_internal_staging`, not `main` — this is a staged fix. The `OldTeams` migration is presumably part of a broader refactor; the diff here is a targeted unblock.

## Verdict

`merge-after-nits` — correct architectural fix (collapse two writers of the same state into one), surgical patch in the production file, test migration is consistent. Two nits: (1) add at least one test that exercises `total_pages > 1` to regression-pin the actual bug class, since `mockTeamsResponse([single team])` only proves the prop-drop didn't break empty/single-page rendering, (2) sanity-grep `app/page.tsx` for residual `teams.` reads that may now be stale-snapshot rather than live.
