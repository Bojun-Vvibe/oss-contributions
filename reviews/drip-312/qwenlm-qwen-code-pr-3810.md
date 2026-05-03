# QwenLM/qwen-code PR #3810 — fix(core): clear FileReadCache on every history rewrite path

- URL: https://github.com/QwenLM/qwen-code/pull/3810
- Head SHA: `aa3b30904f01f1bd096816317754d83d8e249b22`
- Verdict: **merge-as-is**

## Summary

Fixes a stale-cache bug in the Gemini client. The `FileReadCache`
short-circuits `read_file` tool calls when the same file path has
already been read in the current session, on the assumption that the
prior tool result is still in conversation history and therefore the
model has already seen the content. This invariant is broken whenever
the history is mutated out from under it — `resetChat`, `setHistory`,
`truncateHistory`, retry's `stripOrphanedUserEntriesFromHistory`, and
the microcompaction path that strips old `read_file` function responses
to free context. This PR adds `getFileReadCache().clear()` calls on all
of those paths and adds tests for each.

## Specific references

- `packages/core/src/core/client.test.ts:457-535` — five new tests
  covering: `resetChat clears the cache`, `setHistory clears the
  cache`, `truncateHistory clears the cache`, `retry strips orphaned
  trailing user entries and clears the cache`, plus the
  microcompaction-path tests below.
- `packages/core/src/core/client.test.ts:578-680` — two new tests for
  the microcompaction path: `clears the cache after microcompaction
  strips old read_file results` (uses six fabricated `read_file`
  history entries plus a 90-minute idle gap to force the
  `if-meta` branch to fire) and `does not clear the cache when the
  idle gap is below the threshold` (negative case at 30 s of idle
  time, asserting `cacheClear` is *not* called).
- The implementation side (not in this diff slice but inferable from
  the test surface) lives in `packages/core/src/core/client.ts` and
  adds the corresponding `getFileReadCache().clear()` calls in
  `resetChat`, `setHistory`, `truncateHistory`, the retry branch, and
  the microcompaction branch of `sendMessageStream`.

## Commentary

This is an excellent bug-fix PR: it identifies a clear invariant
("FileReadCache assumes the prior content is still in history"),
enumerates every history-mutation surface that violates it, and adds
a test for each. The microcompaction tests in particular are nicely
constructed — `lastApiCompletionTimestamp = Date.now() - 90 * 60_000`
to force the time-based branch, and a paired negative test at
`Date.now() - 30 * 1000` to prove we don't over-invalidate. That
negative test is exactly the kind of guard that prevents the next
"let's just clear the cache everywhere" regression.

A few small observations:

1. **The cache invariant should be documented at the cache definition
   site.** This PR fixes five callers but the underlying invariant
   ("entries are valid only as long as the corresponding
   functionResponse is in history") is implicit. A short doc comment
   on `FileReadCache` itself would let future contributors know they
   need to wire `clear()` into any new history-mutation path they
   add.

2. **`cacheClear` is mocked everywhere via
   `vi.mocked(mockConfig.getFileReadCache).mockReturnValue({ clear:
   cacheClear })`.** That works but means each test has to remember
   to set up the mock. A `beforeEach` that wires a default cache
   stub (and exposes the `clear` spy) would shrink boilerplate. Not
   a blocker.

3. **Retry's `stripOrphanedUserEntriesFromHistory` assertion.** The
   test asserts both `stripOrphanedUserEntriesFromHistory` was
   called *and* `cacheClear` was called, but doesn't assert
   ordering. In practice the order matters: clearing the cache
   *before* the strip would briefly let a duplicate read sneak in.
   Worth a small comment confirming the implementation clears
   *after* the strip.

These are polish items; the PR is correct and comprehensively tested.
Merge as-is.
