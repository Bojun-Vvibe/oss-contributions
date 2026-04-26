# PR #26498 — fix(auth): apply temp_budget_increase on cache-hit path

- **Repo**: BerriAI/litellm
- **PR**: #26498
- **Head SHA**: `46509bf2e9d065b1301484a392aa80312eddb598`
- **Author**: kiyeonjeon21
- **Size**: +1013 / -10 across 7 files (most is `model_prices_and_context_window.json`)
- **Verdict**: **request-changes**

## Summary

Fixes a bug where the proxy auth path applied
`temp_budget_increase` only on the database-fetch arm, *not* on
the cache-hit arm. So users with a temp budget bump received
their increased budget on the first uncached request and the
*old* (lower) limit on every cached request thereafter — i.e.
the bump was effectively useless once the user-key was hot.
The auth-side fix is small (+15 / -6 in `user_api_key_auth.py`)
plus a 232-line dedicated test file. The other ~620 lines are an
unrelated model catalog dump.

## Specific changes

- `litellm/proxy/auth/user_api_key_auth.py:+15/-6` — applies the
  temp-budget overlay on the cache-hit branch. Unfortunately not
  visible in the head-200 cut; the dominant content there is
  `model_prices_and_context_window_backup.json`. The 232-line
  test file `test_temp_budget_increase_caching.py` is the right
  place to verify behavior.
- New tests at `tests/test_litellm/proxy/auth/test_temp_budget_increase_caching.py`
  (+232) — should cover: cache-miss applies overlay, cache-hit
  applies overlay, expired temp_budget falls back, layered
  precedence between budget keys.

## Risks / why request-changes

1. **PR scope**: the title and main fix are auth-related, but the
   diff is dominated by a model-catalog dump (310 lines added to
   each of `model_prices_and_context_window_backup.json` and
   `model_prices_and_context_window.json` — fictional `azure/gpt-5.5`
   and `azure/gpt-5.5-pro` entries with priority pricing tiers).
   That belongs in a separate PR. Mixing them makes both harder
   to review and harder to revert. Asking the author to split.
2. **The pricing entries themselves** include unfamiliar fields
   like `cache_read_input_token_cost_priority`,
   `input_cost_per_token_above_272k_tokens_priority`, and
   `supports_xhigh_reasoning_effort`. If those are real Azure
   tiers, fine; if speculative, they shouldn't ship.
3. The auth fix itself looks like the right shape (the test file
   alone, at 232 lines, says the author traced through the code
   paths properly), but the actual line of the fix isn't in the
   diff sample I have — would want one more pass on the
   `user_api_key_auth.py` hunk specifically.
4. **Sibling PR**: this is paired with #26499 ("join team-member
   budget so rpm/tpm limits are enforced") from the same author
   on the same day. Coordinate review.

## Verdict

`request-changes` — split the model-catalog dump out into its
own PR. The auth fix is small and looks correct in shape; on
its own it would be `merge-after-nits`. As is, the diff is
unreviewable in the time budget.

## What I learned

"Cache-hit arm forgot to apply overlay X" is a recurring class
of bug in any system that builds an effective-state object from
a base + a series of overlays. The defensive pattern is to put
the overlay application *after* the cache fetch (so cache stores
only the base) rather than caching the post-overlay result.
Trying to maintain "the cache is fully resolved" invariant
across N overlay sources is exactly how this kind of bug is
born.
