---
pr: 26503
repo: BerriAI/litellm
sha: aefb54f7a2043bd245e028b6f63a1b68192aa856
verdict: merge-as-is
date: 2026-04-26
---

# BerriAI/litellm #26503 — [Fix] Enforce key.models / user.models on Bedrock passthrough routes

- **Author**: jayy-77
- **Head SHA**: aefb54f7a2043bd245e028b6f63a1b68192aa856
- **Link**: https://github.com/BerriAI/litellm/pull/26503
- **Size**: +193 / -6 across 5 files. Auth-only fix: `auth_utils.py` (+53), `user_api_key_auth.py` (+5/-3), `auth_checks.py` (+1/-1), `factory.py` (+3/-1 black format), and 131 LOC of new tests.

## Scope

Closes a real auth-bypass: `/bedrock/model/{modelId}/{action}` routes carry the model exclusively in the URL path; the native Bedrock InvokeModel/Converse request bodies have no top-level `model` field. `get_model_from_request` only inspected `request_body` and a few OpenAI/Google/Vertex URL patterns, so for Bedrock it returned `None`. Both `_enforce_key_and_fallback_model_access` (key allowlist) and `common_checks` (team/user/project allowlist) skipped `can_key_call_model` when `model is None`, silently letting Bedrock passthrough bypass `key.models` / `user.models`.

## Specific findings

- `auth_utils.py:879-883` — `_BEDROCK_PASSTHROUGH_PATH_RE` covers all six action variants: `invoke`, `invoke-with-response-stream`, `converse`, `converse-stream`, `count_tokens`, `count-tokens`. The optional trailing `/?` is correct (some clients append a trailing slash). The capture group is greedy `(.+)/(...)`, so the `application-inference-profile/{id}` shape (which contains a `/`) is captured intact as one identifier — the test at `:166-172` confirms this. Good.
- `auth_utils.py:887-908` (`_resolve_bedrock_model_id_to_router_model_group`) — scans `llm_router.get_model_list()` for any deployment whose `litellm_params.model` ends with `/{model_id}`. Returns the deployment's `model_name` (the user-facing `model_group` that shows up in `key.models` / `user.models`). Falls back to `None` when no match. Two notes:
  - Linear scan over all router deployments per request. For deployments=O(100) this is negligible; for deployments=O(10k) it's worth caching. Not a blocker.
  - The `endswith(f"/{model_id}")` predicate matches any deployment with that bedrock id, regardless of region/profile prefix. If a user has both `bedrock/global.anthropic.claude-opus-4-7` and `bedrock/us-east-1/global.anthropic.claude-opus-4-7` and routes the request via the bare modelId, the FIRST match wins (iteration order over `get_model_list()`). Probably fine in practice but document the tie-break.
- `auth_utils.py:911-941` — Bedrock branch is gated by `if model is None`, which preserves the existing precedence: explicit body `model` wins, then OpenAI/Vertex URL patterns, then this Bedrock fallback. Test `test_get_model_from_request_bedrock_request_body_model_takes_precedence` confirms. Good.
- `auth_utils.py:919-928` — fallback is `resolved if resolved is not None else model_id`. So unconfigured Bedrock models are returned as the raw id (e.g. `global.anthropic.claude-opus-4-7`), which then hits `can_key_call_model` and gets denied (because raw ids are not in `key.models` for any sensibly configured key). This is the right fail-closed behaviour. Test `test_get_model_from_request_bedrock_falls_back_to_raw_id_when_router_has_no_match` confirms.
- `auth_checks.py:485-487` — `common_checks` now passes `llm_router=llm_router` to `get_model_from_request`. Single-line change. Good.
- `user_api_key_auth.py:872, 1281, 1812` — three call sites updated. Each is wrapping the same `get_model_from_request(request_data, route)` call with the new `llm_router=` kwarg. All three are inside async functions that already have `llm_router` in scope. Verified by inspection.
- Backward compatibility: `llm_router: Optional[Router] = None` is a keyword-default. All non-Bedrock routes are unchanged: the new `_BEDROCK_PASSTHROUGH_PATH_RE.match(route)` only fires when `model is None` AND the route matches the Bedrock pattern. No behavioural drift for any existing route.
- Tests `:255-272` cover: all 6 action variants, application-inference-profile, modelIds with colons (Anthropic versioning) and slashes (cross-account ARNs), router-resolves, no-router-fallback, no-router-no-match, non-action-path-returns-None, request-body-precedence. Eight tests, well-targeted, each tests one rule. PR description shows 64 tests pass in `test_auth_utils.py`, plus 74 in `test_auth_checks.py`, plus 34 in `test_user_api_key_auth.py` — 172 regression-suite tests on the touched files all green.
- `factory.py:5291-5294` — three-line cosmetic black-formatter touch on `default_response_schema_prompt`. Unrelated to the fix but harmless.
- PR description includes before/after curl reproductions showing all four Bedrock passthrough endpoints flipping from HTTP 200 (silent bypass) to HTTP 401 `key_model_access_denied` after the fix. Excellent presentation.

## Risk

Low. This is a security fix (auth bypass) with surgical scope, an opt-in code path (only fires for the documented Bedrock URL pattern), strong fail-closed semantics, and 172 regression tests passing on the touched files. The blast radius is exactly the bug scope.

## Verdict

**merge-as-is** — bug fix is correct, tests are comprehensive, fail-closed behaviour is right, no API breakage. Optional follow-ups: document the deployment-iteration tie-break in `_resolve_bedrock_model_id_to_router_model_group`, and consider caching the deployment-scan if deployment counts grow into the thousands. Neither is a blocker. The unrelated `factory.py` black-formatter chunk could be split into its own commit but is harmless.
