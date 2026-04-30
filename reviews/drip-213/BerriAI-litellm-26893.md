# Review: BerriAI/litellm#26893 — fix: apply cache-read pricing in custom cost path

- PR: https://github.com/BerriAI/litellm/pull/26893
- Head SHA: `13309d173c73e116959224f550de1e063d7e0706`
- Closes: #26807
- Files: `litellm/cost_calculator.py` (+21/-1), `tests/test_litellm/test_cost_calculator.py` (+65/-0)
- Verdict: **merge-after-nits**

## What it does

Fixes a real over-billing bug in the `custom_cost_per_token` code path of
`_cost_per_token_custom_pricing_helper`: when a caller supplies custom pricing that
includes a `cache_read_input_token_cost` rate, the helper was charging *all* prompt
tokens at the full `input_cost_per_token` rate, ignoring the cache-hit discount. After
this PR, cached prompt tokens are subtracted from the uncached portion and charged at
the cheaper rate, mirroring the model-table-driven pricing path.

## Notable changes

- `cost_calculator.py:174-176` — adds two new keyword params to the helper:
  `cache_read_input_tokens: Optional[int] = 0` and `usage_object: Optional[Usage] = None`.
  The default `0` for `cache_read_input_tokens` preserves byte-identical behaviour for
  any caller that doesn't pass it.
- `cost_calculator.py:185-201` — the load-bearing fix. After confirming
  `custom_cost_per_token is not None`, the new code:
  1. Lifts `cache_read_input_token_cost` out via `cast(Optional[float], custom_cost_per_token.get("cache_read_input_token_cost"))`.
     Using `.get(...)` (not `[]`) means the field is genuinely optional — callers that
     don't set it preserve the prior single-rate behaviour through the `else` branch
     at `:200`.
  2. Inside the `if cache_read_input_token_cost is not None:` arm:
     - If `cache_read_input_tokens` was not passed but `usage_object` was, it derives
       cached-token count from `_parse_prompt_tokens_details(usage_object)["cache_hit_tokens"]`.
       This is the right fallback because the existing `cost_per_token` call site
       always has access to `usage_block`, while older external callers may only pass
       the plain `prompt_tokens` integer.
     - `cache_read_tokens = max(0, cache_read_input_tokens or 0)` — defensive against
       negative or `None` values from upstream parsing.
     - `uncached_prompt_tokens = max(0, prompt_tokens - cache_read_tokens)` — defensive
       against the case where reported cache tokens exceed reported prompt tokens
       (clip-to-zero is the right policy; the alternative of going negative would
       produce negative cost).
     - Final `input_cost = full_rate * uncached + cache_rate * cached`.
  - The `else` arm preserves the prior single-rate calculation exactly:
    `input_cost = custom_cost_per_token["input_cost_per_token"] * prompt_tokens`.
- `cost_calculator.py:347-348` — the `cost_per_token` outer fn now threads
  `cache_read_input_tokens=cache_read_input_tokens` and `usage_object=usage_block` into
  the helper. Both values were already in scope at this call site, so no upstream
  plumbing was needed.
- `tests/.../test_cost_calculator.py:51-89` `test_completion_cost_custom_pricing_uses_cache_read_rate`
  — pins the fix with realistic numbers: `prompt_tokens=6074`, `cached_tokens=3456`,
  `completion_tokens=285`, custom rates `2.5e-6 / 1.5e-5 / 2.5e-7`, expected cost
  `pytest.approx(0.011684)`. The math: `(6074-3456)*2.5e-6 + 3456*2.5e-7 + 285*1.5e-5 = 0.006545 + 0.000864 + 0.004275 = 0.011684`. Number checks.
- `tests/.../test_cost_calculator.py:92-117`
  `test_completion_cost_custom_pricing_without_cache_read_rate_preserves_input_rate` —
  pins the no-regression branch: same usage shape, no `cache_read_input_token_cost`
  in the rate dict, expected cost `pytest.approx(0.01946)`. Math:
  `6074 * 2.5e-6 + 285 * 1.5e-5 = 0.015185 + 0.004275 = 0.01946`. Cached tokens are
  ignored (charged at full input rate), which is the documented prior behaviour.

## Reasoning

The bug shape is exactly the kind of silent over-billing that erodes proxy users' trust:
the caller does the right thing (declares both rates in their custom pricing config), the
proxy reports usage with cached-token breakdown, and yet the cost line ignores the
discount because the custom-pricing path never read the cache-rate field.

The fix is at the right layer: the helper that owns the custom-pricing branch. The
alternative "compute cache-discount in `cost_per_token` and pass adjusted prompt_tokens
into the helper" would have made the helper's behaviour depend on whether the caller
pre-adjusted, which is a worse contract.

The two new optional params with safe defaults preserve the helper's external contract
for any caller that imports it directly (the helper appears to be intended-private given
the leading underscore, but Python doesn't enforce that).

The test pair pins both the new branch and the no-regression branch with concrete
numbers; the `pytest.approx` tolerance is the right call for floating-point cost math.

## Nits (non-blocking)

1. `cost_calculator.py:188` — `cache_read_input_token_cost: Optional[float] = cast(...)`.
   The `cast` is a static-analysis hint only; if a misconfigured TOML produces an int
   or a string in this slot, the multiplication on `:198` will silently coerce or
   raise. Worth a single explicit `if not isinstance(cache_read_input_token_cost, (int, float)): raise ValueError(...)`
   guard before the arithmetic, or at least a TODO.
2. `cost_calculator.py:191-194` — the auto-derivation from `usage_object` only fires when
   `cache_read_input_tokens` is *falsy* (`not cache_read_input_tokens`). That means an
   explicit `cache_read_input_tokens=0` from a caller would be overridden by the helper
   reading the usage object. Probably intentional (zero cached tokens is the default
   and uninteresting case) but worth a comment naming the intent so a future reader
   doesn't "fix" it to `is None`.
3. The test for the cache-read-rate branch uses `model="openai/gpt-5.4"` which doesn't
   exist in the standard model registry. That is fine for this test (the custom-pricing
   path bypasses the registry) but a comment naming the intent would help — without it
   a future maintainer might "correct" the model name to a real one and accidentally
   exercise a different code path.
4. No test pins the `prompt_tokens < cache_read_input_tokens` `max(0, ...)` defensive
   clip. Worth one cheap test that constructs that pathological case to lock the
   policy.
5. The tracking-issue link `Fixes #26807` is in the body — good — but there's no
   doc-string update on either `_cost_per_token_custom_pricing_helper` or `cost_per_token`
   describing the new cache-read behaviour. A two-line addition would help future
   readers.
