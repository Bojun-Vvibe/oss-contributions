# Review: block/goose #8916 — fix(bedrock): cache trailing message for stable prefix across agent turns

- **PR**: https://github.com/block/goose/pull/8916
- **Author**: carl-auctane
- **Head SHA**: `00c2141debc4eff86146ed4450ba2249a20ceec2`
- **Base**: `main`
- **Files**: `crates/goose/src/providers/bedrock.rs` (+11/-10)
- **Verdict**: **merge-as-is**

## Reasoning

Relates to #8915. This is a small but cost-impactful fix to Bedrock
prompt-caching placement at `providers/bedrock.rs:232-260`.

The pre-fix logic: when `BEDROCK_ENABLE_CACHING=true`,
`BedrockProvider::converse()` placed cache breakpoints on the **first
three** visible messages (`MESSAGE_CACHE_BUDGET = 3`, then
`idx < cache_count`). The author's analysis identifies why this is
strictly worse than placing the cache point on the trailing message:

- Anthropic / Bedrock prompt caching is keyed by the **hash of the
  prefix ending at the breakpoint**. Reads walk backward up to 20 blocks
  looking for prior writes.
- With the breakpoint fixed at messages 0–2, every turn N reads the same
  cache entry from turn 1, but everything after position 3 is fresh
  bytes that get reprocessed. In an agent loop, the un-cached tail
  grows linearly with turn count.
- With the breakpoint on the trailing message, turn N's lookback finds
  the breakpoint that turn N-1 wrote (provided the new turn added
  fewer than 20 blocks, which is the documented Anthropic-recommended
  pattern for growing conversations).

The diff:

- Replaces the `MESSAGE_CACHE_BUDGET = 3` constant and `cache_count`
  computation with `let last_idx = visible_messages.len().checked_sub(1);
  let cache_last = enable_caching && last_idx.is_some();`.
- The map closure changes from
  `|(idx, m)| to_bedrock_message_with_caching(m, idx < cache_count)`
  to `|(idx, m)| to_bedrock_message_with_caching(m, cache_last &&
  Some(idx) == last_idx)`.
- The misleading `// Cache the earliest messages (not most recent)
  because prompt caching requires exact prefix matching — caching
  recent messages would shift positions each turn, causing misses.`
  comment block is replaced with an accurate one citing the lookback
  semantics and linking the Anthropic docs.

Why this is correct:

- **`checked_sub(1)` correctly handles the empty-conversation edge
  case** — `last_idx` is `None`, `cache_last` is `false`, no breakpoint
  is placed, no panic. Better than the alternative `len() - 1` which
  would underflow.
- **The `cache_last && Some(idx) == last_idx` guard** is semantically
  identical to "if caching is enabled and this is the last message",
  but the explicit `cache_last` short-circuit means the `Option`
  comparison only runs when caching is on. Cleanly composable.
- **No test changes needed**: the existing `to_bedrock_message_with_caching`
  helper tests in `providers::formats::bedrock` already exercise the
  per-message path with `enable_caching=true`, and the
  `test_caching_*` tests in `providers::bedrock` only assert
  `should_enable_caching()` returns the right boolean. Both are
  untouched by this change. Author validated:
  `cargo test -p goose --lib providers::bedrock` (4 passed),
  `cargo test -p goose --lib providers::formats::bedrock` (11 passed),
  plus `cargo clippy --all-targets -- -D warnings` clean.
- **Comment quality is genuinely improved.** The replacement comment
  is operator-grade: explains the cache key, explains the lookback,
  links the canonical doc.

## Suggested follow-ups

- Add a small unit test that asserts cache placement on a 5-message
  conversation: only `idx == 4` should be flagged as cached. Cheap
  insurance against a future refactor that swaps the comparison
  back without realizing.
- The PR mentions an issue-side cost analysis. Worth pulling the
  10-turn cost example into a code-comment or `CHANGELOG.md` entry
  so the cost win is visible at PR-merge time, not buried in the
  linked issue.
- Optional: when conversations exceed the 20-block lookback window
  (rare, but possible in long single-turn tool fan-outs), the
  trailing-only placement degrades. A future enhancement could
  place a *second* breakpoint mid-conversation to extend the
  effective cache reach. Out of scope for this PR.
