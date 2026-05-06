# BerriAI/litellm#27262 — fix(anthropic_messages): strip non-user_id keys from metadata before forwarding

- **Head SHA**: `a05bd278facd2f6398bb2c0cabcc27a13731e8de`
- **Author**: mateo-berri (Mateo Wang)
- **Stats**: +193 / -12, 7 files

## Summary

Mirrors the existing `AnthropicConfig.transform_request` metadata-strip safeguard (PR #24661) into the unified `/v1/messages` flow at `AnthropicMessagesConfig.transform_anthropic_messages_request`. Vertex AI / Azure AI Anthropic 400 with `metadata.tags: Extra inputs are not permitted` on any non-`user_id` key, and the unified Messages path was leaking routing/budget tags. PR also bundles a UI-side change (unified access groups in Add Model dropdown) that author admits is a previously-merged commit riding along.

## Specific citations

- `litellm/llms/anthropic/experimental_pass_through/messages/transformation.py:265-278` is the load-bearing change: extracts `_metadata = anthropic_messages_optional_request_params.get("metadata")`, if dict and `_user_id` non-None replaces with `{"user_id": _user_id}`, else `pop("metadata", None)`. Correct three-way handling (preserve user_id, drop everything else, drop the whole field if user_id missing).
- New test file `tests/test_litellm/llms/anthropic/experimental_pass_through/messages/test_anthropic_messages_metadata_filter.py` — 5 cases at `:23,33,40,46,52` covering: strip non-user_id keys, drop when only disallowed keys, drop when user_id None, pass-through when only user_id, no-op when no metadata field.
- **Off-topic UI bundle**: `ui/litellm-dashboard/src/components/add_model/AddModelForm.tsx:69-71`, `:96-103`, plus `add_auto_router_tab.tsx:39-50`, `edit_auto_router_modal.tsx`, `ModelsAndEndpointsView.tsx:96-119`, and `AddModelForm.test.tsx:104-126` — adds `useAccessGroups()` and merges unified access groups with legacy `model_info.access_groups`. Author flags this in PR body as previously-merged commit `f969eb8c48`.

## Verdict

**request-changes**

## Rationale

The Python fix itself is excellent: minimal, correct three-way logic, comprehensive 5-case unit test, fixes a real customer-facing 400 error class. **But** the PR violates "scope is as isolated as possible" (their own checklist item #2) by carrying ~110 lines of unrelated UI-side access-group merging across 5 frontend files. Author acknowledges the bundling but maintainers should reject and ask for a split — the UI change has its own test (`AddModelForm.test.tsx:301-321`), its own justification ("LIT-2783"), and is unrelated to the Anthropic metadata strip. Reviewer cost is genuine (different reviewers needed: backend Anthropic provider expert vs frontend dashboard engineer), and a bundled merge complicates revert if either half regresses. Once split, the Python half is a fast `merge-as-is`.

