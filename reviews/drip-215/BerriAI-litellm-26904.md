# Review: BerriAI/litellm#26904 — fix: normalize Anthropic server tool usage

- PR: https://github.com/BerriAI/litellm/pull/26904
- Head SHA: `70280b9b272455b2f974d08bc697f67f929755bf`
- State: OPEN
- Files: `litellm/types/utils.py` (+8/-1), `tests/test_litellm/litellm_core_utils/llm_cost_calc/test_tool_call_cost_tracking.py` (+19/-3), `tests/test_litellm/types/test_types_utils.py` (+24/-4)
- Verdict: **merge-after-nits**

## What it does

Fixes a real silent attribute-vs-dict shape mismatch in the Anthropic-passthrough
cost-tracking path. When the proxy's pass-through receives a usage payload that
includes `server_tool_use` as a raw dict (which it does for Anthropic-compatible
responses constructed from the wire), the existing `Usage.__init__` path
assigned `self.server_tool_use = server_tool_use` without normalization, leaving
a `dict` where the type contract promised `ServerToolUse | None`. Downstream
`StandardBuiltInToolCostTracking.response_object_includes_web_search_call(...)`
reads `usage.server_tool_use.web_search_requests` as an attribute, which raises
`AttributeError` on a dict — silently failing cost tracking for Anthropic
server-side web-search calls.

The fix: (1) accept `Optional[Union[ServerToolUse, dict]]` in the constructor
signature, (2) coerce dicts via `ServerToolUse(**server_tool_use)` before
storing, and (3) add `__getitem__` to `ServerToolUse` so callers that already
treated it as a dict (e.g. `usage.server_tool_use["web_search_requests"]`)
continue to work — with a `KeyError` for unknown keys to match dict semantics.

## Notable changes

- `types/utils.py:1525-1528` — new `__getitem__` on `ServerToolUse` raises
  `KeyError(key)` when `key not in self.__class__.model_fields`, then returns
  `getattr(self, key)`. The model-fields check is the right gate because it
  only exposes the declared fields (`web_search_requests`,
  `tool_search_requests`), not arbitrary attributes — so callers can't read
  Pydantic internals via `__getitem__`.
- `types/utils.py:1560` — constructor signature widens from
  `Optional[ServerToolUse]` to `Optional[Union[ServerToolUse, dict]]`. The
  type widening is the only API-surface change; existing callers passing
  `ServerToolUse(...)` are unaffected.
- `types/utils.py:1661-1662` — coercion: `if isinstance(server_tool_use,
  dict): server_tool_use = ServerToolUse(**server_tool_use)`. Critically,
  this happens *before* the `if server_tool_use is not None:` block, so the
  downstream assignment sees a normalized instance. If `**server_tool_use`
  contains unknown keys, Pydantic v2 will raise `ValidationError` by
  default — that's the right strictness for a typed contract, but worth
  confirming Anthropic's wire payload doesn't include any forward-compatible
  fields that would trip this.
- `tests/test_litellm/types/test_types_utils.py:74-96` — new
  `test_usage_converts_server_tool_use_dict` constructs `Usage(...,
  server_tool_use={"web_search_requests": 4, "tool_search_requests": 1})`
  and asserts (a) `isinstance(usage.server_tool_use, ServerToolUse)`, (b)
  attribute access returns `4`, (c) `__getitem__` returns `4`, (d)
  `usage.server_tool_use["unknown_metric"]` raises `KeyError`, and (e)
  round-trip through `Usage(**usage.model_dump())` preserves the type and
  values. The round-trip pin is the load-bearing assertion — it pins that
  serialization → deserialization keeps `ServerToolUse` shape, which is the
  exact path that triggered the original bug.
- `tests/test_litellm/.../test_tool_call_cost_tracking.py:139-153` — new
  `test_get_cost_for_anthropic_web_search_with_server_tool_use_dict`
  constructs `Usage(server_tool_use={"web_search_requests": 1})` and calls
  `StandardBuiltInToolCostTracking.response_object_includes_web_search_call(
  response_object=None, usage=usage)`. This is the regression repro — pre-fix,
  this would raise `AttributeError: 'dict' object has no attribute
  'web_search_requests'`. Post-fix, returns truthy.

## Reasoning

The bug class here is "shape contract enforced at one boundary but not the
other." `ServerToolUse` was a Pydantic model in the type system and a dict
on the wire, with the conversion happening in some entry points but not the
constructor itself. Anthropic's pass-through path constructs `Usage` from a
raw response dict (because the response shape varies enough across providers
that early normalization was avoided), so it hit the un-normalized arm.

The chosen fix is the right shape: normalize at the constructor boundary so
*every* entry point produces a typed instance, rather than chasing call
sites and adding ad-hoc dict-handling in each cost-tracking helper. The
`__getitem__` addition is defensive — it lets existing dict-shaped callers
continue working without changes, which matters because there's likely
unaudited downstream code (custom callbacks, integrations) that already
treats `server_tool_use` as a dict-like by inspection.

The two new tests cover both the cause (dict input → typed output) and the
symptom (web-search cost lookup succeeds with dict input). The
`response_object=None` argument in the symptom test is intentional — it
forces the helper to read from `usage.server_tool_use` rather than from a
response-object-level field, isolating the failure mode.

## Nits (non-blocking)

1. `types/utils.py:1525` — `__getitem__` raises `KeyError(key)` but the
   message would be more useful as `KeyError(f"ServerToolUse has no field
   {key!r}")` to distinguish from generic dict misses in stack traces.
2. `types/utils.py:1661` — `ServerToolUse(**server_tool_use)` will raise
   `ValidationError` (subclass of `ValueError`) on unknown keys with default
   Pydantic v2 strictness. If Anthropic ever adds a third counter field
   (`computer_use_requests`, etc.), every downstream consumer breaks until
   the litellm types catch up. Consider `ServerToolUse.model_validate(
   server_tool_use)` with `model_config = ConfigDict(extra="ignore")` on the
   `ServerToolUse` class so forward-compat fields are silently dropped.
3. The new tests don't pin behaviour for the unknown-key case at the
   constructor boundary (only at the `__getitem__` boundary). A test like
   `Usage(server_tool_use={"unknown_metric": 1})` with the expected
   `ValidationError`/`pass-through-preserves-known-only` semantics would
   document the intent.
4. The PR title says "normalize Anthropic server tool usage" but the change
   is provider-agnostic — any caller passing a dict benefits. Title could
   be "fix: normalize server_tool_use dict in Usage constructor" for
   precision.
5. `__getitem__` covers reads but not `__contains__`. `"web_search_requests"
   in usage.server_tool_use` would still return `False` for a typed
   instance. If dict-like callers also use `in`-tests, add `__contains__`
   for parity.
