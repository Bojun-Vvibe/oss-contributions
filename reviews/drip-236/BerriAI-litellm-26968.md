# BerriAI/litellm#26968 — chore(proxy): tighten router-settings-override and mock-testing trust

- **PR**: https://github.com/BerriAI/litellm/pull/26968
- **Head SHA**: `e60a72ee1de40e97b48354b29d8cca856e810289`
- **Size**: +362 / -17, 39 files (most are auto-generated next.js HTML route renames)
- **Verdict**: **merge-after-nits**

## Context

VERIA-44 internal security finding. Two closely-related bypass paths in proxy:

1. **`router_settings_override.fallbacks` bypass.** `route_llm_request.py` lifts `router_settings_override.{fallbacks, context_window_fallbacks, content_policy_fallbacks}` into per-request kwargs *after* the auth-side model-access check at `_enforce_key_and_fallback_model_access`. Today only top-level `request_data["fallbacks"]` is walked by the auth check, so any model placed in the nested override slips through and gets executed against the router with the API key's allowlist already cleared.
2. **`mock_testing_*` flag forwarding.** `Router._handle_mock_testing_fallbacks` exists for router unit tests to deterministically force fallback execution by raising synthetic `InternalServerError` / `ContextWindowExceededError` / `ContentPolicyViolationError`. It pops the flags directly from kwargs. When a request body containing these flags is forwarded to the router via the proxy, the router honors them. Combined with #1 above, this turns "theoretical fallback bypass" into "deterministic execution against any restricted model" — the attacker can choose which fallback class fires.

## What's right

**`user_api_key_auth.py:2141-2199` — fallback-validation widened to all three fields × both surfaces.**

Old code (deleted at `:2141-2169`):
- `fallback_models = cast(Optional[List[ALL_FALLBACK_MODEL_VALUES]], request_data.get("fallbacks", None))` — single field.
- Loop over `fallback_models`, calling `can_key_call_model(model=m["model"] if isinstance(m, dict) else m, ...)` and `is_valid_fallback_model(...)`.
- Walked only top-level, not `router_settings_override`.

New code:
- `fallback_names: List[str] = []`
- Loop over `ROUTER_FALLBACK_FIELDS = ("fallbacks", "context_window_fallbacks", "content_policy_fallbacks")` at `:2173`.
- For each field, extend with `iter_router_fallback_model_names(request_data.get(_fb_key))` (top-level surface) AND `iter_router_fallback_model_names(override_settings.get(_fb_key))` (nested `router_settings_override` surface).
- `for _name in dict.fromkeys(fallback_names):` — iterate deduplicated, order-preserving.
- Each unique name through `can_key_call_model` and `is_valid_fallback_model`.

**The `dict.fromkeys()` dedup is the right choice** — preserves insertion order (so error messages reference the first occurrence), avoids the overhead of redundant DB hits for the same model name appearing in both surfaces, and is the canonical Python idiom (vs `list(set(...))` which loses order).

**`iter_router_fallback_model_names` helper at `:2185-2199`** handles both shapes the codebase uses:

- Simple top-level: `[str | {"model": str}]` — for each entry, yield `entry` if str, or `entry["model"]` if dict.
- Router-config nested: `[{primary: [fallback_models]}]` — for each entry that's a dict with a non-`"model"` shape, iterate its values; for each value that's a list, yield each str/`{model: str}` element.

The shape disambiguation happens correctly: the dict.get(`"model"`) check at `:2196-2198` handles the simple `{"model": str}` shape and **falls through with `continue`** to the router-config branch only if `"model"` isn't present. This prevents a router-config-shaped entry like `{"gpt-4": [{"model": "claude"}], "model": "synthetic"}` from being interpreted as the simple shape and silently dropping the nested fallbacks.

**`route_llm_request.py:9-19, 337-342` — `mock_testing_*` strip at the proxy boundary.**

New module-level constant `_MOCK_TESTING_KWARG_NAMES = ("mock_testing_fallbacks", "mock_testing_context_fallbacks", "mock_testing_content_policy_fallbacks")` with the right hardcoded-with-test-pin discipline: PR body explains why it's hardcoded rather than `dataclasses.fields(MockRouterTestingParams)` at import time — `litellm.types.router` imports back into proxy modules before this module finishes loading, so deriving the list dynamically would create a circular import. The accompanying test `test_mock_testing_kwarg_names_matches_dataclass` (referenced in the constant's comment) keeps the hardcoded list in lockstep with the dataclass definition. This is the **exactly correct trade-off**: structural test-time enforcement of a runtime-hardcoded list when import-time enumeration is infeasible.

The strip at `:337-342` happens inside `route_request(...)` immediately after `await add_shared_session_to_data(data)` and before any router dispatch. Test code that calls the router directly (bypassing proxy `route_request`) is unaffected — the right scope, since the legitimate test surface lives at the router's own kwargs interface.

**Test coverage:**
- `tests/test_litellm/proxy/test_route_llm_request.py` — parametrized test asserting all three `mock_testing_*` flags are stripped before reaching the router.
- `tests/test_litellm/proxy/auth/test_router_override_fallback_auth.py` — 7 tests covering helper shape handling and auth-side validation across all three override fallback fields. The new file (visible at the diff tail) is the exact regression pin for VERIA-44.

## Risks / nits

- **`override_settings = request_data.get("router_settings_override")`** at `:2167` reads the *unparsed* request body. If `router_settings_override` is wrapped in another field (e.g., `request_data["metadata"]["router_settings_override"]`) by some middleware not visible in this diff, the validation would miss it. Worth a one-line confirmation in PR body that the unwrapping happens *before* this auth hook (typically yes, but the code should say so).
- **`isinstance(override_settings, dict)` guard at `:2169`** silently no-ops if `router_settings_override` is malformed (e.g., a list). Combined with the loose Pydantic shape used elsewhere in proxy this is defensible (fail-safe = ignore rather than crash on auth path), but a `WarningEvent`-style log for "router_settings_override has unexpected shape, skipping fallback validation" would help operators spot misconfigured client SDKs that are sending bad shapes.
- **`iter_router_fallback_model_names` doesn't handle the `Pydantic` shapes some clients send** (e.g., a `Fallback` model object with a `.model` attribute). If clients can send Pydantic models (which Pydantic-aware FastAPI deserializers may produce when the schema permits), the helper's `isinstance(entry, dict)` branch would skip them. Worth a one-line confirmation that `request_data` is always plain dict by the time the auth hook fires (typically yes — JSON body → dict — but the code should pin this).
- **`_MOCK_TESTING_KWARG_NAMES` is a `tuple:` annotation without parameterization** at `:16`. Should be `Tuple[str, ...]` (already imported at the top of the module per the proxy convention). Cosmetic.
- **The hardcoded list synchronization test is mentioned in the comment but not visible in the diff.** Worth confirming `test_mock_testing_kwarg_names_matches_dataclass` actually exists and runs as part of the standard suite — if it's missing, a future `mock_testing_*` flag added to `MockRouterTestingParams` would silently bypass the strip.
- **39 files changed, ~32 are auto-generated next.js route renames** (`out/foo.html` → `out/foo/index.html`). These are unrelated to the security fix and are the by-product of a Next.js build artifact regeneration. Worth splitting the security fix from the build-artifact PR for cleaner audit trail and revert path.
- **`#noqa: PLR0915 - Complex routing function`** at the top of `route_request` already says the function is past pylint's complexity budget; this PR adds 6 more lines to it. Worth a follow-up refactor to extract the new strip into a `_strip_router_internal_kwargs(data: dict) -> None` helper colocated with the constant.
- **Auth-time validation is the right defense layer, but the runtime strip at `route_request` adds defense in depth.** The PR has both — good. A third layer (router-side `_handle_mock_testing_fallbacks` itself rejects when called from a non-test context) would close the loop completely, but the current two-layer fix is sufficient for VERIA-44 closure.

## Verdict

**merge-after-nits.** Real security fix with the right two-layer architecture: auth-time validation widened to cover all three fallback fields × both top-level/nested surfaces, plus a runtime strip of `mock_testing_*` flags at the proxy→router boundary so an attacker can't deterministically choose which fallback class fires. The hardcoded-with-test-pin discipline for `_MOCK_TESTING_KWARG_NAMES` is exactly right given the circular-import constraint. Address the build-artifact split, the malformed-override-warning log, and the `Tuple[str, ...]` annotation before merge.
