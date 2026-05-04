# Review: block/goose #8916

- **Title:** fix(bedrock): cache trailing message for stable prefix across agent turns
- **Head SHA:** `00c2141debc4eff86146ed4450ba2249a20ceec2`
- **Scope:** +11 / -10 single file `crates/goose/src/providers/bedrock.rs`
- **Drip:** drip-338

## What changed

Replaces the previous "cache the first 3 visible messages" strategy with
"cache the last visible message". The old approach was inverted relative
to how Bedrock prompt caching actually works: caching the earliest
messages does indeed give a stable prefix, but it caps cached content at
3 messages forever. The new approach places the cache breakpoint on the
trailing message; on the next turn, Bedrock's lookback walks back from
the new trailing position and finds the previous turn's cache entry, so
each turn only re-processes the delta since the last request.

## Specific observations

- `crates/goose/src/providers/bedrock.rs:232-248` — the comment is
  rewritten to explicitly cite the prompt-caching docs and explain why
  trailing-message caching works ("cache entries are keyed by the hash
  of the prefix ending at the breakpoint; on each new turn, the lookback
  walks backward from the new trailing position and finds the previous
  turn's entry"). This is the right mental model and the comment will
  age well.
- `crates/goose/src/providers/bedrock.rs:240-242` — `let last_idx =
  visible_messages.len().checked_sub(1)` correctly returns `None` for an
  empty `visible_messages` slice, and `cache_last = enable_caching &&
  last_idx.is_some()` short-circuits the empty case. Better than the
  previous `min(MESSAGE_CACHE_BUDGET)` which still tried to cache zero
  messages cleanly but read awkwardly.
- `crates/goose/src/providers/bedrock.rs:250-256` — the per-message
  closure now reads
  `to_bedrock_message_with_caching(m, cache_last && Some(idx) ==
  last_idx)`. The `Some(idx) == last_idx` comparison against
  `Option<usize>` works because `usize == usize` lifts cleanly through
  `Option`'s `PartialEq`. Idiomatic.
- The previous `MESSAGE_CACHE_BUDGET: usize = 3` constant is removed —
  good, no dead code left.
- No test changes in the diff. For a behavior change in the caching
  layout, an integration test that asserts the cache breakpoint position
  on a 5-message conversation would meaningfully prevent regression.
  Worth flagging as a follow-up if the existing bedrock test suite
  doesn't cover this directly.

## Risks

- Behavior change: any user relying on the previous "cache first 3
  messages" semantics for some workload will see a cache invalidation
  on the next turn. In practice the new behavior reduces re-processing
  cost across the board, so the regression risk is bounded to cost
  observability dashboards (one-time discontinuity).
- Bedrock's documented cache-point limits (typically 4 breakpoints per
  request) are not exceeded — this PR only places one breakpoint.
- The linked URL `https://platform.claude.com/docs/en/build-with-claude/prompt-caching`
  should be confirmed as the canonical Anthropic docs path; the
  `platform.claude.com` host is correct as of late 2025 but worth a
  spot-check.

## Verdict

**merge-after-nits** — correct fix with a much-improved comment; ask
for a regression test that pins the cache breakpoint position on a
multi-turn conversation before merging.
