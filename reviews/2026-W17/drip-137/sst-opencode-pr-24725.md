# sst/opencode #24725 — fix(tui): sort session picker by full updated timestamp

- PR: https://github.com/sst/opencode/pull/24725
- Head SHA: `dd8d2dd9865c`
- Diff: 18+/7- across 1 file (`packages/opencode/src/cli/cmd/tui/component/dialog-session-list.tsx`)
- Base: `dev` · Closes #24727

## Verdict: merge-as-is

## Rationale

- **Right fix for a real "recently-updated session is hard to find" symptom.** The pre-diff `options()` memo at `dialog-session-list.tsx:113-119` sorted via `new Date(b.time.updated).setHours(0, 0, 0, 0) - new Date(a.time.updated).setHours(0, 0, 0, 0)`, i.e. day-bucket primary key with `b.time.created` tiebreaker — so a session updated 5 minutes ago could rank below one updated this morning if both fell in the same calendar day, and *worse*, an active session updated minutes ago but created days back would sort by its created-date ancestor and bury under same-day creations. The new `orderByRecency()` helper at `:112-117` collapses to a single `b.time.updated - a.time.updated` comparator and drops the calendar-day bucketing entirely. Today's `today` const at `:120` is now unused — minor dead-code follow-up but doesn't block.
- **The "stable order while open, recompute on open" pattern is the load-bearing structural choice.** The PR correctly identifies that re-sorting on every memo recomputation would cause the row the user is *aiming for with arrow keys* to jump to the top mid-keystroke. The fix at `:118` introduces `[browseOrder, setBrowseOrder] = createSignal<string[]>([])` captured once via `setBrowseOrder(orderByRecency(sync.data.session))` inside the existing `onMount` at `:185`, then `options()` at `:122-128` looks up live `sessionMap` rows by frozen ID list — so workspace status, busy spinner, and other per-row data still update reactively while the *positions* stay pinned. This is exactly the contract from the prior stabilization PR #22987 (referenced in the body) preserved with strictly better recency-on-open behavior.
- **Search-mode branch is symmetric and correct.** At `:124-125`, `searchResult ? orderByRecency(searchResult) : browseOrder()` means typing into search returns a freshly-sorted-by-recency result list, but once that list is in hand, the same frozen-positions invariant holds (because `searchResults()` is itself a memo that only changes when the query string changes, not when individual session timestamps tick). So a user searching "auth" while a background `auth-debug` session ticks won't have it jump positions either.
- **`sessionMap` rebuild on every memo run is acceptable.** At `:121`, `new Map(sessions().filter(...).map((x) => [x.id, x]))` is rebuilt per memo invocation — but the memo runs only when `sessions()` (or `searchResults()`) changes, not per render, and the picker is bounded to whatever the API page returns. Negligible cost vs the alternative of maintaining a separate signal-driven map.

## Nits / follow-ups

- The unused `const today = new Date().toDateString()` at `:120` should be deleted in this PR — leaving it in invites the next reader to think it's load-bearing.
- The `setBrowseOrder` is captured at `onMount`, which means if the dialog stays open for hours and many sessions get created/updated, the new sessions won't appear in `browseOrder` (they will appear in `sessionMap` but the lookup-by-frozen-ID skips them). Worth documenting at `:185` that "frozen ordering" intentionally means new sessions during the dialog's lifetime are invisible until reopen — or alternatively, append-new-IDs-to-tail on `sync.data.session` change while preserving existing positions.
- No test cell. The dialog-session-list component has no co-located test, so the recency-vs-stability invariant has no regression anchor. Worth a tiny snapshot test that asserts `options()` order after a background `time.updated` mutation matches the pre-mutation order.

## What I learned

The "stable display order while open, recompute on open" pattern shows up across UI-list components anywhere arrow-key navigation matters: capture an `orderByX(rows)` snapshot into a signal at mount/open, do live-data lookups by frozen ID, only re-snapshot on explicit signals (search query change, dialog re-open, manual refresh). The previous PR #22987 had got the stability half right but with too coarse a primary sort key (calendar day) that defeated the recency intent; this PR keeps the stability contract and tightens the sort key to the natural one. The `sessionMap` rebuild-vs-maintain trade is the canonical "memos are cheaper than you think" lesson.
