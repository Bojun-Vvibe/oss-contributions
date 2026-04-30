# BerriAI/litellm #26831 — fix(proxy): inherit caller identity in passthrough batch managed-object

- **URL:** https://github.com/BerriAI/litellm/pull/26831
- **Head SHA:** `2461139593a4749439af9dae7d4e36e3958908fd`
- **Merge SHA:** `7c8e95888a71c09b5b3963cf7b5ee851a3dd633d`
- **Files:** `litellm/proxy/pass_through_endpoints/llm_provider_handlers/anthropic_passthrough_logging_handler.py` (+8/-3 at `_store_batch_managed_object` `:549-565`), `litellm/proxy/pass_through_endpoints/llm_provider_handlers/vertex_passthrough_logging_handler.py` (+8/-3, identical pattern at `:849-865`), `tests/test_litellm/proxy/pass_through_endpoints/llm_provider_handlers/test_anthropic_passthrough_logging_handler.py` (+51 lines, parameterized regression at `:573-628`), `tests/test_litellm/proxy/pass_through_endpoints/test_vertex_ai_batch_passthrough.py` (+51 lines, parallel regression at `:264-319`)
- **Verdict:** `merge-as-is`

## What changed

Fixes a real identity-leak bug in batch-managed-object storage on the passthrough path: `_store_batch_managed_object` was constructing the fabricated `UserAPIKeyAuth` from `kwargs.get("user_id", "default-user")` and `team_id=None` — but the **upstream pass-through caller doesn't put `user_id` at the top of `kwargs`**; it puts it in `kwargs["litellm_params"]["metadata"]["user_api_key_user_id"]` (the standard place identity-bearing fields live throughout the rest of the proxy). Result: every Anthropic and Vertex batch managed-object was being stored under `"default-user"` with `team_id=None`, regardless of who actually authenticated the request.

The fix at `anthropic_passthrough_logging_handler.py:552-561` (and the identical change at `vertex_passthrough_logging_handler.py:852-861`):

1. Extracts metadata defensively: `_request_metadata = (kwargs.get("litellm_params", {}) or {}).get("metadata", {}) or {}` at `:552-554`. The double `or {}` guards against both `litellm_params` being `None` (not just absent) and `metadata` being `None`. Standard belt-and-suspenders for a path that takes external dict shapes.
2. Reads `user_id` from `_request_metadata.get("user_api_key_user_id", "default-user")` at `:557-559` — preserves the existing `"default-user"` fallback for the truly-unauthenticated edge case (e.g., management endpoints or test fixtures).
3. Reads `team_id` from `_request_metadata.get("user_api_key_team_id")` at `:561` — defaults to `None` when absent, which preserves the existing behavior for un-team-scoped requests.

The 51-line parameterized regression at `test_anthropic_passthrough_logging_handler.py:573-628` and the matching one at `test_vertex_ai_batch_passthrough.py:264-319` lock both branches: the metadata-present case asserts `user_id == "real-user-123"` and `team_id == "team-456"`, the metadata-absent case asserts the `"default-user"` / `None` fallback. Both tests assert `mock_managed_files_hook.store_unified_object_id.assert_called_once()` plus the `user_api_key_dict.user_id` / `.team_id` fields on the captured call, which is the right shape — locks the *observable side effect* on the storage hook, not the implementation detail of how the dict is constructed.

## Why it's right

- **The bug is a real privacy/billing leak.** Storing every batch under `"default-user"` means (a) per-user spend attribution on batches is wrong, (b) per-team spend attribution is wrong, (c) the managed-object lookup-by-user path can't find batches for the actual originator, and (d) audit logs lose the identity association. This isn't theoretical — `assert call_kwargs["user_api_key_dict"].user_id == expected_user_id` proves the production path was returning `"default-user"` before the fix.
- **The metadata shape is the canonical one in this codebase.** `litellm_params.metadata.user_api_key_user_id` / `user_api_key_team_id` is the standard place identity lives across the proxy — every other auth-aware code path reads from there. Aligning the passthrough batch handler with the rest of the codebase is a strict improvement: one fewer special case for maintainers to remember.
- **The double-`or {}` defensive shape at `:552-554` matters here.** Pass-through callers come from `anthropic_passthrough` and `vertex_passthrough` request flows that have evolved independently and may legitimately pass `litellm_params=None` in some test/edge-case paths. The previous `kwargs.get("user_id", ...)` shape was tolerant of any kwargs shape; preserving that tolerance in the new code path means the fix doesn't introduce new `AttributeError` failure modes.
- **Parameterized tests cover both the present and absent cases for both providers** — that's the minimum to lock both branches against regression. Two providers × two cases = four parameterized assertions, all green; if either provider's metadata-extraction shape drifts, the test fails immediately.
- **Test asserts `assert_called_once()` AND the field values** — that's the right shape because it locks both "the side effect happened" and "the side effect carried the correct payload." Either alone would be insufficient: `assert_called_once()` without field assertions wouldn't catch the bug being fixed; field assertions without call-count would silently pass if the hook was called multiple times with different dicts.

## No nits

The diff is minimal (8 lines per provider for the production fix), the fix is in the right place (the construction site for `UserAPIKeyAuth`, not at any of the call sites that consume the stored object), the regression coverage is symmetric across both providers and both branches, and the assertions lock the observable behavior at the storage hook. Ship it.
