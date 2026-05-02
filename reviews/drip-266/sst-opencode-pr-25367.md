# sst/opencode #25367 — fix(session): cache messages across prompt loop to preserve prompt cache byte-identity

- **Repo:** sst/opencode
- **PR:** #25367
- **URL:** https://github.com/sst/opencode/pull/25367
- **Head SHA:** `0724daf5fd5aa33ccc86d937ed7372ab32d0665e`
- **Files touched:**
  - `packages/app/vite.js` (+2 -1) — vite alias for `@opencode-ai/core`
  - `packages/opencode/src/session/prompt.ts` (+16 -1) — message cache
- **Verdict:** merge-after-nits

## Summary

The prompt loop previously re-ran `MessageV2.filterCompactedEffect(sessionID)`
at the top of every iteration, returning a freshly-allocated array each
turn. Even though the contents were equivalent, the new object identity
(plus any incidental ordering nondeterminism in the underlying read)
defeated upstream prompt-cache hashing on providers that key by byte
identity of the serialized message array. This PR caches `msgs` across
iterations and only fully reloads when the loop re-enters from a
state-changing branch (subtask, instruction overflow, compaction).

## Specific notes

- **`packages/opencode/src/session/prompt.ts:1283-1298`** — the
  `needsFullReload` flag is set `true` initially and after every branch
  that mutates session state (`handleSubtask`, instruction completion,
  compaction.create, instruction overflow at line 1505). Logic is
  correct: any branch that `continue`s after side-effects on the session
  forces a reload.
- **Lines 1289-1296** — the incremental merge path uses `m.info.id` for
  dedup and only appends genuinely new messages. This preserves the
  identity of prior message objects (which is what the caching here is
  for). Good.
- **`packages/app/vite.js:9-13`** — the indentation churn (4-space →
  6-space on `resolve:`) muddies the diff. The actual change is just the
  one alias entry. Reformat or split into two commits if the maintainer
  cares.

## Nits

- Consider asserting in dev that `fresh.length >= msgs.length` after the
  merge — if the underlying `filterCompactedEffect` ever shrinks (e.g.
  external compaction not flagged via `needsFullReload`), the cache will
  silently retain stale `msgs` references.
- The `let msgs: MessageV2.WithParts[] | undefined` annotation is
  load-bearing but not commented; a one-liner explaining "cached across
  loop iterations to preserve prompt-cache identity" would help future
  readers.
