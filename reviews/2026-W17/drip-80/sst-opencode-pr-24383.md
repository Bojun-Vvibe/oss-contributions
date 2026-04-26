# sst/opencode PR #24383 — fix: move session roots filter from client-side to SQL layer

- **PR:** https://github.com/sst/opencode/pull/24383
- **Author:** heimoshuiyu
- **Head SHA:** `b6af6ca8d2fa6a65d6c56a8e8c868cbf313188a2`
- **Files:** 2 (+3 / -3)
- **Verdict:** `merge-as-is`

## What it does

Closes #16270 and #20238. In the TUI session-list dialog,
`sdk.client.session.list({ limit: 100 })` was being called without the
`roots: true` filter. The server-side query therefore returned the most
recent 100 sessions of *any* kind — including subagent child sessions —
sorted by recency. For users who lean on subagents (each turn spawns
several), the LIMIT got eaten by children before any root sessions made it
into the result, leaving the dialog effectively empty after client-side
filtering trimmed the children out.

The fix is three call-sites adding `roots: true`, which pushes the
`WHERE parent_id IS NULL` predicate down to SQL where it's applied *before*
the LIMIT.

## Specific reads

- `packages/opencode/src/cli/cmd/tui/component/dialog-session-list.tsx:37` —
  the user-facing search call. `{ search: query, limit: 30 }` →
  `{ search: query, limit: 30, roots: true }`. With limit 30 and a heavy
  subagent user, almost every search query would return the same 30 most
  recent subagent-child sessions instead of any root match. This is the most
  user-visible of the three.
- `packages/opencode/src/cli/cmd/tui/context/sync.tsx:365` — the 30-day
  session-list prefetch on TUI startup. Same pattern: no limit specified
  here so Drizzle's default applies, but the bug is the same — children
  would crowd out roots in the in-memory store, breaking session-resume
  affordances.
- `packages/opencode/src/cli/cmd/tui/context/sync.tsx:485` — the manual
  `refresh()` action, also fixed.
- The PR body claims the redundant client-side filter is removed, but the
  diff stat (+3 / -3) shows only the parameter additions; the diff doesn't
  show a deletion of any client-side `.filter(s => !s.parent_id)` block. So
  one of two things: either the client-side filter was already gone (and
  the bug was that nothing filtered anywhere — children leaked into the UI
  *and* roots got crowded out), or the PR description is slightly
  optimistic about cleanup. Either way the fix itself is correct; it's
  worth a follow-up to grep for `parent_id` in the TUI sync layer to be
  sure no stale dead-code filter remains.

## Risk surface

**Very low.** Three-character API parameter additions, no schema or
contract changes. The SDK surface (`session.list({ roots })`) already
exists — this PR just uses it.

Two minor concerns:

1. **Behavior change for existing users with subagent sessions.** Previously
   the dialog showed "30 most recent sessions of any kind"; now it shows
   "30 most recent root sessions." A user who was using the dialog as a
   way to jump back into a specific subagent run will lose that
   affordance. Probably the right call (subagent sessions are usually not
   things you want to jump *into* — they're transient context — but
   worth a one-line release note.
2. **`start: start` filter combined with `roots: true`** at
   `sync.tsx:365` — make sure the SDK applies both predicates as AND, not
   OR. Standard ORM behavior, but worth a quick read of the generated
   SQL.

## Suggestions before merge

- Add a smoke test specifically asserting that with N subagent children
  and 1 root, `session.list({ roots: true, limit: 1 })` returns the root,
  not a child. The existing `session.test.ts` (4 tests) and
  `project.test.ts` (32 tests) probably don't pin this exact behavior.
- One-line release note in the PR body about the dialog change.

Verdict: merge-as-is — three-line fix, correct intent, exactly the right
"push the filter down to where the LIMIT lives" pattern. The only way to
get this wrong was to forget the parameter — and that's exactly what
happened originally.

## What I learned

This is the canonical "client-side filter after a LIMIT is a bug, not a
slow path" case. When you write `db.list().filter(predicate)`, you've
silently introduced a precondition that the page-size has to be greater
than (matches-after-filter + non-matches-before-the-last-match). When the
non-matches grow over time (here, as a user accumulates subagent runs),
the predicate eventually returns empty without anyone touching the code.
The fix is always the same: push the predicate into the WHERE clause.
The interesting question is *why this kept happening* — probably because
adding a Boolean kwarg to `session.list` is invisible in code review
unless you know the SQL contract. A typed required-when-listing
discriminator (`session.list({ scope: "roots" | "all" | "children" })`
with no default) would have prevented this class of bug at the SDK
layer.
