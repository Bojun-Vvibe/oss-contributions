# QwenLM/qwen-code #3848 — fix(memory): route auto-memory recall selector to fast model

- **Head SHA:** `55df20b27d1123ca9f38eb6a731d400e9535e970`
- **Base:** `main`
- **Author:** B-A-M-N (John London)
- **Size:** +52 / −1 across 2 files
  (`packages/core/src/memory/relevanceSelector.ts`,
  `packages/core/src/memory/relevanceSelector.test.ts`)
- **Verdict:** `merge-as-is`

## Summary

`selectRelevantAutoMemoryDocumentsByModel()` previously dispatched its
side query to the default session model. This PR pipes
`config.getFastModel()` into the `runSideQuery` call so the
auto-memory recall — a background, latency-sensitive, throwaway
classification task — runs on the fast model when one is configured.
Falls back to the session model when no fast model is set.

## What's right

- **The change is exactly two lines of behavior + one comment.**
  `relevanceSelector.ts:94-96`:
  ```
  // Use the fast model for this background side-query to reduce latency and
  // cost. Falls back to the main session model if no fast model is configured.
  model: config.getFastModel(),
  ```
  The comment tells you *why*, not *what*, which is what review
  comments are for. Easy to grep for, easy to revert if the fast
  model produces lower-quality recall.

- **Fallback semantics are correct.** `config.getFastModel()` returns
  `string | undefined`. When it returns `undefined`, `runSideQuery`
  presumably falls through to the session default — confirmed by the
  test at `relevanceSelector.test.ts:108-128` (`passes undefined model
  when no fast model is configured`) which asserts the call gets
  `model: undefined`. So users who haven't configured a fast model
  see no behavior change.

- **Tests pin both branches.**
  - `passes the fast model to runSideQuery when configured`
    (lines 81-105) — sets `getFastModel` to return `'fast-flash-model'`
    and asserts `runSideQuery` is called with that exact string.
  - `passes undefined model when no fast model is configured`
    (lines 108-128) — opposite branch.
  Both also assert `purpose: 'auto-memory-recall'` and `config: {
  temperature: 0 }` are preserved, so the side query's other
  invariants don't drift.

- **Mock setup is the right shape.** `relevanceSelector.test.ts:42-44`
  changes `mockConfig` from `{} as Config` to
  `{ getFastModel: vi.fn().mockReturnValue(undefined) }`. The default
  return is `undefined`, so existing tests that don't care about the
  fast model still see the original behavior. Per-test
  `vi.mocked(mockConfig.getFastModel).mockReturnValue(...)`
  selectively overrides for the new tests. Clean.

- **Auto-memory recall is the correct workload for fast-model
  routing.** This is a classification task ("which of these N memory
  docs are relevant to the user query?") with a structured-output
  schema (`RESPONSE_SCHEMA`), `temperature: 0`, and a 5-second timeout
  (`AbortSignal.timeout(5_000)`). Exactly the workload profile that
  benefits most from a faster, cheaper model — quality difference is
  small, latency difference is large. Same reasoning as gemini-cli
  #26498's steering ack and many other "background side-query"
  patterns.

## Risk

- **Quality regression on memory recall:** if the configured fast
  model has noticeably worse instruction-following on the
  `selected_memories` JSON schema, recall quality drops. Mitigation:
  the existing `validate?` callback in `runSideQuery` (referenced at
  `relevanceSelector.test.ts:131-138`) already throws on unknown
  relative paths, so a fast model that hallucinates filenames will
  fail loudly rather than silently degrade. Worth a pre-merge spot
  check on whichever fast model Qwen Code typically pairs with
  (e.g., `qwen-flash` or equivalent).
- **No quality regression for default users:** `getFastModel()` is
  presumably opt-in, and the test `passes undefined model when no
  fast model is configured` confirms the fallback. Users on default
  config see zero behavior change.

## Nits

None worth blocking on.

- The new comment block at `relevanceSelector.ts:93-96` is two lines
  of comment for one line of code — a bit verbose, but the rationale
  ("background side-query to reduce latency and cost") is exactly the
  context a future maintainer needs when wondering whether to delete
  the line.
- The PR could also add `purpose: 'auto-memory-recall'` to the new
  test assertions explicitly (it does — `expect.objectContaining({
  purpose: 'auto-memory-recall' })`). Already done.

## Verdict

`merge-as-is` — minimal, correct, and well-tested. The kind of
boring two-line perf fix that's easy to review and easy to land.
