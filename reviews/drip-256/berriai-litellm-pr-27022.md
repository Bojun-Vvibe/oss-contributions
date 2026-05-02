# BerriAI/litellm PR #27022 — feat: selectively apply routing strategy according to model name

- URL: https://github.com/BerriAI/litellm/pull/27022
- Head SHA: `3b443d4dfd869a87364460605cec714c9f3788fa`
- Author: yassin-berriai
- Verdict: **request-changes**

## Summary

Adds a new `routing_strategy_model_filter: Optional[List[str]]` to `Router`, `RouterConfig`, and `UpdateRouterConfig`. When set, the configured `routing_strategy` only applies to deployments whose `model_name` is in the list; other models fall through to `simple-shuffle`. Implemented via a new `_get_effective_routing_strategy(model)` helper that's substituted for `self.routing_strategy` checks in both `async_get_available_deployment` and `get_available_deployment`.

## Line-level observations

- `litellm/router.py` line 311 + 545: the new constructor kwarg and instance attribute are wired in straightforwardly. Default `None` preserves existing behavior — good.
- Lines ~9618–9633: `_get_effective_routing_strategy` is a one-line predicate. Correct, but **falls back to literal `"simple-shuffle"` rather than to a configured fallback strategy**. For users who want, say, `latency-based-routing` for `gpt-4*` and `usage-based-routing-v2` for everything else, they're stuck — the fallback is hard-coded.
- Lines ~9678–9683 in `async_get_available_deployment`: `effective_strategy` is computed *after* the `pre_routing_hook`. Good — the comment makes this explicit: "the hook can replace `model` and the filter must key off the final model name."
- Sync path (lines ~10093–10130) in `get_available_deployment` also uses `_get_effective_routing_strategy(model)`. Consistent with the async path.
- **Bug**: the `least-busy` branch (lines 9748–9751 in async) was `self.routing_strategy == "least-busy" and self.leastbusy_logger is not None` — the rewrite kept the conjunction but if `effective_strategy == "least-busy"` and `leastbusy_logger is None`, the chain falls through silently to the trailing else (which I can't see in the truncated diff but historically returns `None` or raises). Same as before, but if the filter forces `least-busy` for a model when the logger isn't initialized, you'll get a confusing error far from the misconfiguration. Consider validating in `__init__` that `routing_strategy_model_filter` only contains models the configured strategy can actually serve.
- Tests file `tests/test_litellm/router_strategy/test_routing_strategy_model_filter.py` (221 lines, only first lines visible) is new — good coverage for the basic in-list / out-of-list split, but I'd want to see a test for the post-`pre_routing_hook` model rewrite case explicitly.

## Major concerns

1. **Naming**: `routing_strategy_model_filter` reads like "filter the list of models this strategy considers." It actually means "list of models this strategy applies to." Consider `routing_strategy_models` or `routing_strategy_applies_to_models`.
2. **Hard-coded fallback to `simple-shuffle`** is too restrictive. Users likely want either (a) configurable fallback, or (b) per-model strategy mapping (`routing_strategy_overrides: Dict[str, str]`), which is a strict superset of this feature.
3. **Inverted logic also valid**: some users want "use simple-shuffle for these specific models, but the fancy strategy everywhere else." This API can't express that.

## Suggestions

1. Rename to something less ambiguous (`routing_strategy_models` or similar).
2. Add an optional `routing_strategy_fallback: Optional[str] = "simple-shuffle"` so the fallback is configurable.
3. Better still, replace the whole feature with a per-model strategy override dict — same code surface, much more expressive.
4. Add a test exercising `pre_routing_hook` model rewrite to lock in the documented "filter applies to post-hook model" semantics.
