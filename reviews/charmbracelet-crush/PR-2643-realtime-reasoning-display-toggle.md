# crush PR #2643 — fix: enable real-time reasoning display and implement missing toggle handler

Link: https://github.com/charmbracelet/crush/pull/2643

## What it changes

Two related bugs in the TUI reasoning surface:

1. **Stream rendering broken.** `RawRender()` cached output keyed
   only by terminal width. The first render captured an empty
   reasoning block; every subsequent frame hit the cache and
   returned the same stale empty content, so streamed reasoning
   tokens never appeared. Fix: bypass the cache while
   `isSpinning()` (i.e., a turn is in flight); fall back to the
   cache after the turn completes.
2. **Toggle handler missing.** The command palette already had a
   "Hide/Show Thinking" entry but no implementation, so selecting
   it was a no-op. Fix: add a `showThinking` bool to the UI struct
   (default `true`), implement `ActionToggleThinkingVisibility`,
   add a `refreshMessages()` helper, thread the flag through
   `ExtractMessageItems` / `NewAssistantMessageItem`, and only
   render the thinking block when `showThinking`.

64 added / 17 deleted across UI / chat / dialog packages.

## Critique

The stream-rendering fix is the more interesting bug: a cache keyed
on a stable input dimension (width) where the *content* is the thing
that changes is a textbook stale-cache class, and the current code
only worked accidentally — for non-reasoning models the first render
already had final content, so the cache was never wrong. Bypassing
the cache during `isSpinning()` is the correct shape.

Two concerns:

1. **`isSpinning()` is a sloppy proxy for "content may still
   change."** A turn that is in flight but currently in a tool-call
   round-trip may have a stable assistant message displayed; we
   would still re-render every frame, which on slow terminals
   visibly re-flows. A tighter predicate — "current message has an
   active streaming reasoning channel" — would avoid the per-frame
   cost when only tool calls are in flight. Acceptable to land the
   coarser fix and tighten later, but the comment should call this
   out so the next perf pass knows to look here.
2. **`showThinking` default and persistence.** The PR defaults to
   `true` and adds no persistence. That means every new session
   starts with thinking visible, even for users who toggled it off
   last time. For a feature explicitly added because it is noisy
   for some users, the default + non-persistence combination is
   the worst of both worlds. Either persist the toggle in the
   user's TUI config, or default to `false` and let users opt in.
3. **Cache key.** A more durable fix would key the cache on
   `(width, contentHash)` rather than special-casing the
   spinning state. That way streaming, post-stream, and
   late-arriving updates (e.g., a redraw triggered by a resize
   while a tool call updates state) all behave correctly without
   a state-machine predicate.

## Suggestions

- Persist `showThinking` to the TUI config / restore on startup.
- Add a TODO to swap `isSpinning()` for a content-hash cache key
  in a follow-up.
- Add a regression test that streams reasoning tokens and asserts
  the rendered frame changes between frames during streaming.

Verdict: real bug, correct cure. Land with persistence + a test.
