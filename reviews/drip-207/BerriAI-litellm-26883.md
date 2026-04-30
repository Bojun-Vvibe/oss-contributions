# BerriAI/litellm #26883 — fix(cost): apply cache pricing for custom token costs

- Head SHA: `be73bf1472a76e1f3293ad1490ced2d514c04153`
- Files: `litellm/cost_calculator.py`, `litellm/types/utils.py`, `tests/test_litellm/test_custom_pricing_cache_cost.py`
- Size: +133 / -3

## What it does

Closes #26807 (and is the natural symmetric companion to #26872
which fixed the same bug on the *automatic* pricing path; this
one fixes it on the *user-supplied* `custom_cost_per_token`
path). When a caller passes
`custom_cost_per_token={"input_cost_per_token": ..., "output_cost_per_token": ..., "cache_read_input_token_cost": ...}`,
pre-PR `_cost_per_token_custom_pricing_helper` ignored both the
cache-read pricing key and the cache-hit token count, billing
every prompt token at the regular `input_cost_per_token` rate —
which silently overbilled cached requests by the difference,
typically a 10× factor on Anthropic-style cache reads.

The fix:

- `CostPerToken` TypedDict (`litellm/types/utils.py:111-115`)
  gains two `NotRequired[float]` fields:
  `cache_creation_input_token_cost` and
  `cache_read_input_token_cost`. `NotRequired` is the right
  call here — these are opt-in pricing knobs, and existing
  callers passing only the two original fields keep type-
  checking.
- `_cost_per_token_custom_pricing_helper` at
  `cost_calculator.py:172-218` gains a new
  `usage_object: Optional[Usage]` kwarg, defaulted to `None` so
  pre-existing callers don't break.
- The new helper body at `:184-214` does:
  1. Read `cache_read_input_token_cost` and
     `cache_creation_input_token_cost` from the custom pricing
     dict, defaulting each to `input_cost_per_token` (so an
     unconfigured cache field falls back to the regular rate
     — which preserves the pre-PR behavior bit-for-bit when no
     cache pricing is supplied).
  2. If a `usage_object` was passed, extract cached/created
     token counts via two paths: first
     `_parse_prompt_tokens_details(usage_object)` for the
     OpenAI-shaped `prompt_tokens_details.cached_tokens`
     pathway, then a `getattr(usage_object,
     "cache_read_input_tokens"/"cache_creation_input_tokens", None)`
     fallback for the Anthropic-shaped flat fields. The
     `cache_read_tokens or (...)` chain uses `or` to prefer
     the OpenAI path when both are populated.
  3. Compute `uncached_prompt_tokens = max(0, prompt_tokens -
     cache_read_tokens - cache_creation_tokens)` — the `max(0,
     ...)` is defensive against pathological providers that
     report `cached_tokens > prompt_tokens`.
  4. Bill: `uncached × input + cache_read × cache_read_cost +
     cache_creation × cache_creation_cost`.
- `cost_per_token` at `:359` threads the existing `usage_block`
  through to the helper.

Three regression tests in
`tests/test_litellm/test_custom_pricing_cache_cost.py` exercise
the three pathways:

- `test_custom_cost_per_token_uses_cache_read_pricing` — exact
  numbers from issue #26807: 6074 prompt tokens with 3456 cached,
  custom pricing of $2.50/M input and $0.25/M cache-read, asserts
  `cost == pytest.approx(0.011684)`. The arithmetic checks out:
  `(6074 - 3456) × 0.0000025 + 3456 × 0.00000025 + 285 × 0.000015
  = 0.006545 + 0.000864 + 0.004275 = 0.011684`.
- `test_custom_cost_per_token_uses_cache_read_usage_field` —
  validates the Anthropic-shaped `cache_read_input_tokens` flat
  field path with explicit arithmetic in the assertion:
  `(60 * 0.01) + (40 * 0.001) + (10 * 0.02)`.
- `test_custom_cost_per_token_uses_cache_creation_pricing` —
  validates `cache_creation_input_tokens` with similarly explicit
  arithmetic.

## What works

The `NotRequired` annotation on the two new `CostPerToken` fields
is exactly the right type-system call: opt-in pricing knobs that
shouldn't break existing callers, and `mypy --strict` will still
catch typos like `cache_read_input_token_cost_per_token`.

Defaulting `cache_read_input_token_cost` to
`input_cost_per_token` when unset (`cost_calculator.py:185-189`)
is the correct preservation of pre-PR behavior: a caller who
doesn't supply cache pricing gets billed at the regular input
rate for cached tokens — same as before. Only callers who *did*
supply `cache_read_input_token_cost` see different numbers, and
they see the correct numbers.

The dual extraction path (OpenAI's
`prompt_tokens_details.cached_tokens` AND Anthropic's flat
`cache_read_input_tokens`) is necessary — different providers
populate different shapes, and a user supplying custom pricing
shouldn't have to know which shape their proxy is producing.

`max(0, prompt_tokens - cache_read_tokens - cache_creation_tokens)`
is defensive in the right way. The OpenAI usage shape allows
`cached_tokens > prompt_tokens` in some edge cases (cached items
that were dropped from the actual prompt window), and without
the clamp `uncached_prompt_tokens` would go negative and emit a
*credit*. `max(0, ...)` makes the failure mode "we billed the
cache rate twice and the regular rate zero times", which is
cheaper for the user and harder to spot but at least never
negative.

The test-side arithmetic is spelled out in the assertions
(`(60 * 0.01) + (40 * 0.001) + (10 * 0.02)`) rather than
hardcoded as `0.62`, which means a future contributor reading
the test can see *why* the expected number is what it is — much
better than a magic constant.

The companion to #26872 framing in the PR's positioning is
correct — these two PRs together close the cache-pricing-leak
across both the model-DB pricing path and the user-supplied
pricing path.

## Concerns

1. **`cache_read_tokens or (getattr(...) or 0)` short-circuits on
   zero.** At `cost_calculator.py:198-200`, the fallback chain is
   `cache_read_tokens = cache_read_tokens or (getattr(...) or 0)`.
   If the OpenAI path successfully extracts
   `cached_tokens = 0` (a valid value: "no cache hit happened
   on this request"), the `or` triggers the fallback to the
   Anthropic field, which is wrong — we should trust the
   OpenAI-path zero, not silently fall back. Fix: extract the
   OpenAI path into a sentinel like
   `if usage_object.prompt_tokens_details is not None: ... ; else:
   fallback`. The visible test data all has nonzero cached
   tokens so this isn't caught.
2. **`custom_cost_map = cast(dict[str, float], custom_cost_per_token)`
   is a load-bearing type-system shrug.** The `cast` is necessary
   because `CostPerToken` is a TypedDict with two required `float`
   fields and now two `NotRequired[float]` fields, but `dict[str,
   float]` doesn't capture the optionality. The downstream
   `custom_cost_map.get(key, default)` call works but loses the
   compile-time guarantee that `key` is one of the four valid keys
   — a typo wouldn't be caught. Better: `custom_cost_per_token.get
   ("cache_read_input_token_cost", input_cost_per_token)` directly,
   which works because TypedDicts support `.get`.
3. **No test for both cache_read AND cache_creation in the same
   request.** Anthropic-shaped responses commonly have both
   nonzero (cache miss on first turn → creation tokens; subsequent
   turns → read tokens, but a single mid-conversation request can
   have both). The math should be straightforward — both paths are
   independent — but a fourth test combining them would lock down
   the addition order and confirm the `cache_read_tokens or` chain
   doesn't accidentally cross-contaminate.
4. **`usage_block` propagation is silent.** The change at
   `cost_calculator.py:359` adds `usage_object=usage_block` to
   the helper call, but `cost_per_token` has many call paths and
   the comment doesn't note that `usage_block` is now load-bearing
   for cache-aware billing. A docstring tweak on
   `_cost_per_token_custom_pricing_helper` noting "pass
   `usage_object` to enable cache-aware pricing" would help the
   next caller.
5. **Output cost still ignores cache.** Output tokens have no
   cache concept (you don't cache generated output), so this is
   correct, but the test naming
   (`test_custom_cost_per_token_uses_cache_read_pricing`) doesn't
   explicitly assert the output cost was billed at the regular
   rate — the `pytest.approx(0.011684)` aggregate captures it but
   a future test reader has to do the arithmetic to confirm. Not
   blocking.

Verdict: merge-after-nits

## What I learned

The `or`-chain shape `x = primary_path or fallback_path` is the
classic Python idiom for "prefer A, fall back to B" but it has a
sharp edge: it treats *any falsy value* (zero, empty string,
empty list) the same as "absent". For a token count where zero is
a meaningful answer ("there were definitely no cache hits, I
checked"), the `or` collapses that into "I have no information,
try B" — which silently changes billing semantics. The safer
shape is to use a sentinel (`None` is the canonical one) and
explicitly check `if primary_path is None: fallback`, even though
it's three lines instead of one. Worth remembering whenever the
"prefer A else B" decision involves a numeric or container type
where zero/empty is semantically distinct from missing.
