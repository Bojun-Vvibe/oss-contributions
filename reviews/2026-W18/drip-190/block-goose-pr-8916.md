# block/goose #8916 — fix(bedrock): cache trailing message for stable prefix across agent turns

- **URL:** https://github.com/block/goose/pull/8916
- **Head SHA:** `00c2141debc4eff86146ed4450ba2249a20ceec2`
- **Files:** `crates/goose/src/providers/bedrock.rs` (+11/-10)
- **Verdict:** `merge-after-nits`

## What changed

`BedrockProvider` was placing the prompt-cache breakpoint on the *first three messages* of the conversation (`MESSAGE_CACHE_BUDGET = 3`, caching `idx < cache_count`). The author's stated rationale at the previous comment was correct in spirit ("cache earliest because prompt caching requires exact prefix matching") but produced *zero cache hits in practice* on Anthropic's Bedrock implementation, because Bedrock's cache is keyed by *prefix-ending-at-the-breakpoint*, and a breakpoint at index 2 means the cache entry covers messages [0..2] — which never grows. Every subsequent turn re-pays for messages [3..N], scaling linearly with conversation length.

The fix at `bedrock.rs:235-256`:
- Replaces `MESSAGE_CACHE_BUDGET = 3` and the `idx < cache_count` check with a `last_idx = visible_messages.len().checked_sub(1)` and `cache_last && Some(idx) == last_idx` placement.
- The cache breakpoint now sits on the *trailing* message every turn. Because Bedrock's `lookback` walks backward from each new trailing position, it will hit the *previous turn's* trailing-message breakpoint as a prefix, so fresh tokenization is bounded to "content added since the last request" rather than "content after position 3".
- Updates the comment block at `:235-242` with a Claude-platform doc URL pointing at the prompt-caching reference.

## Why it's right

- The diagnosis matches the documented Anthropic prompt-cache behavior: cache entries are keyed by the SHA of the prefix-ending-at-each-breakpoint, and lookback walks backward from each *new* breakpoint position. Anchoring to the trailing message means turn N's breakpoint at message N-1 will subsume turn N-1's breakpoint at message N-2 as part of its prefix, getting cache hits all the way back. The previous "cache the first 3" anchored to a static prefix that everything-after-3 always missed.
- The implementation is minimal and correct: `cache_last && Some(idx) == last_idx` evaluates to `true` for exactly one message per request (the last visible one), which respects Bedrock's per-request breakpoint budget (Bedrock allows up to 4 cache breakpoints, this PR uses 1).
- `enable_caching && last_idx.is_some()` correctly handles the empty-`visible_messages` case (no caching when there's nothing to cache), and the `checked_sub(1)` is the right defense against `visible_messages.len() == 0` underflow.
- The comment update is appropriately specific — names the hash-of-prefix-ending-at-breakpoint mechanism + lookback-walks-backward semantics + the "fresh processing bounded to content added since the last request" outcome. Future maintainers can audit whether the contract still holds without re-reading the Bedrock SDK source.

## Nits / not blockers

- **No regression test in this diff.** The `to_bedrock_message_with_caching` adapter is presumably tested elsewhere, but a focused unit test that builds a `visible_messages` of length 5 and asserts only `idx == 4` gets `cache_breakpoint=true` (and `idx == 0..=3` gets `false`) would lock the contract. Without it, a future refactor that flips back to `idx < N` could re-introduce the bug silently. The git-blame trail will show the rationale, but a test is stronger.
- **The `last_idx.is_some()` branch is redundant with `Some(idx) == last_idx`.** When `last_idx == None`, the comparison `Some(idx) == None` is always `false` in the closure regardless, so the outer `cache_last` guard is structurally a "skip the iteration overhead" optimization. Could be simplified to:
  ```rust
  let last_idx = if enable_caching { visible_messages.len().checked_sub(1) } else { None };
  // ... inside the map:
  to_bedrock_message_with_caching(m, Some(idx) == last_idx)
  ```
  Minor style point — the explicit `cache_last` flag is fine and arguably more readable.
- **`visible_messages` filter at `:233-234` is by reference but the trailing message identity used for caching depends on `is_agent_visible()` ordering.** If a future change introduces a *non-visible* trailing message (e.g. a hidden tool-result placeholder), the cache breakpoint silently shifts to the last *visible* one. That's correct behavior for caching but worth a comment naming "trailing visible, not absolute trailing" so a future contributor doesn't break the invariant by reading "last message" naively.
- **The doc URL `https://platform.claude.com/docs/en/build-with-claude/prompt-caching`** is canonical for Anthropic prompt caching but Bedrock's behavior is the *Bedrock implementation* of that contract — there's a non-zero chance Bedrock's caching semantics drift from the canonical doc. Adding a second link to the AWS Bedrock prompt-caching announcement / docs would shore up "this is how Bedrock specifically implements it" for future maintainers chasing a divergence.
- **`MESSAGE_CACHE_BUDGET = 3` constant deletion** is fine but if this provider ever wants to use multiple breakpoints (Bedrock allows up to 4) you'll be reintroducing a similar constant. A `CACHE_BREAKPOINTS_PER_REQUEST: usize = 1` named-thing in the new comment would make that future expansion cheaper.

## Risk

Low. The behavioral change is "cache the trailing message instead of the first three", which under the documented Bedrock cache semantics is strictly an improvement: the previous behavior cached zero useful entries; the new one caches a growing prefix. Worst case if the analysis is wrong, the cache simply doesn't hit (today's behavior). No correctness contract on the model output changes — caching is purely a billing/latency optimization.
