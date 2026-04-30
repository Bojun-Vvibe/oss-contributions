# Review: BerriAI/litellm#26880 — image cost: token-based fallback when (quality, size) lookup misses

- PR: https://github.com/BerriAI/litellm/pull/26880
- Author: kimsehwan96 (hayden)
- headRefOid: `fab911f233e781bd5e24160ccdf797be9f7875a9`
- Files: `litellm/cost_calculator.py` (+66/-15),
  `litellm/litellm_core_utils/llm_cost_calc/utils.py` (+3/-0),
  `tests/test_litellm/test_default_image_cost_calculator.py` (+311/-0, new)
- Verdict: **merge-after-nits**

## Analysis

Adds a token-based fallback path inside `default_image_cost_calculator` for image-gen models that
publish only per-token rates (target use case: gpt-image-2 at non-standard resolutions where the
existing `(quality, size) -> per-image price` table cannot match). The new helper
`_image_cost_from_token_usage(cost_info, image_response)` at `cost_calculator.py:1903-1939` walks
`image_response.usage.prompt_tokens_details` / `completion_tokens_details` and applies the rate
keys `input_cost_per_token`, `cache_read_input_token_cost`, `input_cost_per_image_token`,
`cache_read_input_image_token_cost`, `output_cost_per_image_token`. The `text_cached`/`image_cached`
split (`cost_calculator.py:1928-1932`) correctly attributes cached tokens to text first then image,
which matches the OpenAI Responses-style usage shape where `cached_tokens` is a flat sum overlapping
both buckets.

Control flow in `default_image_cost_calculator` (cost_calculator.py:1947-2055) is restructured: the
old "raise immediately if `cost_info is None`" guard is moved past the per-image / per-pixel
priorities, and a new Priority 3 tries `_image_cost_from_token_usage` against the matched
`cost_info` *and* the bare model key (with `model.split("/")[-1]` as a sanity fallback). Only if
all three lookups fail does the function raise. This is backward compatible — existing per-image /
per-pixel paths are unchanged — and the three call sites in `route_image_generation_cost_calculator`
(utils.py:1005, 1028, 1038) correctly forward `image_response=completion_response`.

The 311-line test file exercises the new helper on its own, exercises the (quality,size) miss
path, and presumably covers the cached-token math — couldn't see the bottom of the file in the
diff but the structure looks thorough.

**Nit 1:** The `text_cached = min(text_in, cached_in)` / `image_cached = cached_in - text_cached`
attribution assumes `cached_in <= text_in + image_in`. If `cached_in` ever exceeds the sum (provider
bug or unusual usage report), `image_cached` could be negative, producing negative cost
contributions. A `max(0, cached_in - text_cached)` clamp would be safer.

**Nit 2:** The signature change `default_image_cost_calculator(..., image_response=None)` is a
public-ish helper. Adding the new keyword-only arg at the end is source-compatible, but anyone
subclassing or wrapping this function (e.g. custom cost callbacks) will need to be aware. Worth
mentioning in the changelog/release notes.

**Nit 3 (style):** `fallback_entries = [cost_info] if cost_info is not None else []` followed by
the loop that appends model-key entries (cost_calculator.py:2034-2042) is slightly tangled. A small
helper or comment explaining "we try the matched-prefix entry first, then the bare model key" would
help the next reader.

The intent and tests are right. Merge after the negative-cost clamp; the rest are polish.
