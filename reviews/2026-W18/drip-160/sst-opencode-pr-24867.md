# sst/opencode PR #24867 — fix(app): improve sidebar session load more behavior

- Repo: `sst/opencode`
- PR: https://github.com/sst/opencode/pull/24867
- Head SHA: `414738410a59860970ac3d30233e578bf5cfde16`
- State: OPEN, +86/-7 across 6 files

## What it does

Two coupled UX/correctness fixes for the workspace sidebar's session list:

1. **Bigger first page**: bumps the initial `limit` per-directory store from `5` → a new `SESSION_ROOT_PAGE_SIZE = 20` constant (`types.ts:134`, `child-store.ts:213`).
2. **Honest "Load More"**: replaces the buggy cache-reuse predicate `meta.limit >= store.limit` with a four-input `canReuseRootSessionCache({cachedRootCount, fetchedLimit, requestedLimit, total})` helper (`session-load.ts:27-37`) that also counts how many *root* (non-archived, non-child) sessions are actually in the store, not just what the server-reported `limit` was.

## Specific reads

- `session-load.ts:27-37` — the new predicate:
  ```ts
  export function canReuseRootSessionCache(input: {
    cachedRootCount: number
    fetchedLimit: number
    requestedLimit: number
    total: number
  }) {
    if (input.fetchedLimit < input.requestedLimit) return false
    if (input.cachedRootCount >= input.requestedLimit) return true
    return input.total <= input.cachedRootCount
  }
  ```
  The third clause is the load-bearing one: it handles the "we asked for 40, server only has 20 root sessions, all 20 are already cached" case which the old `meta.limit >= store.limit` mishandled (would refetch redundantly). Test at `global-sync.test.ts:107-115` pins exactly this case.

- `global-sync.tsx:212-215` — the cached-root-count derivation that feeds the predicate:
  ```ts
  const cachedRootCount = store.session.filter(
    (s) => !!s?.id && !s.parentID && !s.time?.archived,
  ).length
  ```
  Three-clause filter (`id` truthy, no parent, not archived) correctly mirrors the canonical "root session" definition used elsewhere. **Risk**: this is a linear scan over `store.session` on every `loadSessions(directory)` call for a pinned directory; for users with thousands of cached sessions per workspace, this is hot-path. A counter maintained at insertion/deletion time would eliminate the scan, but that's a follow-up.

- `sidebar-workspace.tsx:328` and `:460` — the load-more handlers are now uniform:
  ```ts
  setWorkspaceStore("limit", nextRootSessionLimit)  // both SortableWorkspace and LocalWorkspace
  ```
  where `nextRootSessionLimit(limit) => (limit ?? 0) + SESSION_ROOT_PAGE_SIZE` (was `+ 5`). Test at `global-sync.test.ts:91-94` pins both the initial-from-undefined case and the additive case.

- Test surface (`global-sync.test.ts:86-115`) covers the three meaningful branches of `canReuseRootSessionCache`: cache-covers-limit (true), trimmed-cache-cannot-satisfy (false), all-roots-cached-but-trimmed (true). The last test is named slightly misleadingly — `cachedRootCount: 20, requestedLimit: 40, total: 20` describes "server has only 20 roots total" not "trimmed cache."

## Verdict: `merge-after-nits`

## Rationale

This is a structurally correct fix to a real bug — the previous `meta.limit >= store.limit` predicate conflated "we fetched at least this many" with "we have at least this many cached," which fell apart whenever the cache had been trimmed by `trimSessions` or whenever the server had fewer roots than requested. The new four-input predicate is auditable and the tests pin the load-bearing branches. Two small concerns to address before merge: (a) the `cachedRootCount` filter is recomputed on every `loadSessions` call and is O(n) over the cached session list — fine today at the new 20-page size but worth a comment if sessions are expected to grow large per directory, and (b) the third test case's name `"reuses cache when all available roots are already cached"` should explicitly say "even when the historical fetch over-asked," since that's the bug it's pinning. The page-size bump from 5 → 20 also deserves a note on the perf implication for cold-load on directories with many sessions (4× the initial server fetch).
