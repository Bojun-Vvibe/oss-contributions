---
pr: block/goose#8916
sha: 00c2141debc4eff86146ed4450ba2249a20ceec2
verdict: merge-as-is
reviewed_at: 2026-04-29T18:31:00Z
---

# fix(bedrock): cache trailing message for stable prefix across agent turns

## Context

In `crates/goose/src/providers/bedrock.rs` (`BedrockProvider::converse`,
around line 232), the previous code placed prompt-cache breakpoints on
the first three visible messages
(`const MESSAGE_CACHE_BUDGET: usize = 3; let cache_count = … visible_messages.len().min(MESSAGE_CACHE_BUDGET)`)
and then iterated `enumerate()` setting cache=true for `idx < cache_count`.
This PR replaces that with a single trailing-message cache point.

The author's reasoning matches Anthropic's prompt caching contract:
cache reads walk *backward* from the breakpoint, hashing the prefix.
A cache point pinned to position 0..3 means everything appended after
position 3 has to be reprocessed every turn — linear growth in turn
count. A trailing breakpoint means the next turn's lookback (≤20
blocks) finds the previous turn's write, and only the new content
between turns gets fresh processing.

## What's good

- The diff is exactly the change described in the comment — no
  drive-by refactors. The new `last_idx = visible_messages.len().checked_sub(1)`
  + `cache_last && Some(idx) == last_idx` pattern is the cleanest
  Rust expression of "set the flag for exactly the last element,
  or none if the list is empty."
- The misleading old comment ("caching recent messages would shift
  positions each turn, causing misses") is replaced with an accurate
  description of the lookup model and a documentation link. Future
  maintainers won't re-introduce the original mistake based on that
  comment.
- The author correctly identified that the existing test surface
  (`providers::formats::bedrock` per-message helpers and
  `providers::bedrock::test_caching_*` enable-flag tests) doesn't
  actually exercise the *placement* of the cache point — it exercises
  whether `to_bedrock_message_with_caching` honors the boolean.
  Adding a placement test would be valuable but is not strictly
  required for correctness; the change itself is mechanically obvious.
- The system-prompt cache point is left untouched, which is correct
  — system prompts are stable across turns, so a head-anchored
  breakpoint is the right policy there.

## Concerns / nits

- For agent loops that add >20 blocks per turn (large multi-step
  tool batches), the trailing-breakpoint strategy degrades to
  full-prefix reprocessing because the lookback window is exceeded.
  This is documented behavior, not a bug, but worth a comment in
  the code or a follow-up that emits a debug log when this
  threshold is approached.
- `enable_caching && last_idx.is_some()` could be folded into a
  `let cache_last = enable_caching && !visible_messages.is_empty();`
  for one less `Option` ceremony. Style nit only.

## Verdict

`merge-as-is` — the analysis is correct, the diff is minimal, the
old comment was wrong and is now right. The 20-block-window
caveat is a follow-up consideration.
