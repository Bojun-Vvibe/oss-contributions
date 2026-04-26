---
pr: 26546
repo: BerriAI/litellm
sha: a4ffdacebf0973a9d0c0edfef0d67d217f17e44c
verdict: merge-after-nits
date: 2026-04-26
---

# BerriAI/litellm #26546 — feat(guardrails): add peyeeye PII redaction & rehydration guardrail

- **Author**: tim-peyeeye
- **Head SHA**: a4ffdacebf0973a9d0c0edfef0d67d217f17e44c
- **Link**: https://github.com/BerriAI/litellm/pull/26546
- **Size**: ~720 diff lines: new `litellm/proxy/guardrails/guardrail_hooks/peyeeye/{__init__,peyeeye}.py`, new `litellm/types/proxy/guardrails/guardrail_hooks/peyeeye.py`, `litellm/types/guardrails.py` updates, plus 222-line test file.

## Scope

Vendor-contributed guardrail integration: pre-call hook redacts PII tokens (e.g. emails → `[EMAIL_1]`) before messages hit the model, post-call hook rehydrates the placeholders back to original values in the model's reply. Two session modes — `stateful` (peyeeye stores token→value mapping under a `ses_…` id) and `stateless` (returns a sealed `skey_…` blob, no server-side retention).

## Specific findings

- `peyeeye/peyeeye.py:74-90` — credentials resolved via constructor → `api_key` kwarg → `PEYEEYE_API_KEY` env, in that order. Standard pattern. Raises typed `PeyeeyeGuardrailMissingSecrets` on miss. Good.
- `peyeeye/peyeeye.py:130-160` (`async_pre_call_hook`) — extracts text from each message via `_iter_message_text` (handles both `content: str` and `content: [{type: "text", text: ...}, ...]` multimodal shape), batches them in one `/v1/redact` call, mutates `messages` in place, caches the returned `session_id` under key `peyeeye_session:{litellm_call_id}` with 3600s TTL. Cache-key derivation falls back to `id(data)` if `litellm_call_id` is missing — this is a footgun: `id(data)` is a Python object id that won't match between pre-call and post-call invocations if `data` is reconstructed (it likely is). When `litellm_call_id` is absent, redaction effectively becomes one-way and PII never gets rehydrated in the response. Recommend either failing closed (raise) or logging a warning so users don't silently ship `[EMAIL_1]` to end users.
- `peyeeye/peyeeye.py:215-240` (`async_post_call_success_hook`) — note this reads from `litellm.cache` (the global), not from the `cache: DualCache` argument that the pre-call hook wrote to. There's an asymmetry: pre-call writes to `cache.set_cache(...)` (the per-request DualCache), post-call reads from `litellm.cache.get_cache(...)`. If `litellm.cache` is unconfigured (which is the default unless the user set up Redis/in-memory cache explicitly), every redaction will silently fail to rehydrate. The test `test_post_call_rehydrates_response` at `:594` works only because it monkey-patches `litellm.cache = MagicMock()`. **This needs to either consistently use the DualCache from the request context or have a fallback path.**
- `peyeeye/peyeeye.py:250-258` — `_rehydrate` swallows exceptions and returns the un-rehydrated text. That's the right call for availability (don't break the response if peyeeye is down) but combined with the cache-asymmetry bug above, users could ship `[EMAIL_1]` placeholders to production end-users without noticing. A loud `verbose_proxy_logger.warning` is logged but it's only at `warn` level.
- `peyeeye/peyeeye.py:260-262` — `_delete_session` is fire-and-forget DELETE on `/v1/sessions/{id}` with 10s timeout. Reasonable cleanup.
- `peyeeye/peyeeye.py:280-300` (`_reraise_api_error`) — typed-error mapping for 401 → `PeyeeyeGuardrailMissingSecrets`, 429 → rate-limit, timeout → typed timeout. Clean. Good.
- `peyeeye/peyeeye.py:331-358` (`_iter_message_text` / `_set_message_text`) — only handles `role/content`-shaped messages with text or `type:"text"` parts. Does NOT handle `type:"image_url"`, `type:"input_audio"`, tool-call args, or `tool_result` content. PII embedded in image alt-text, tool arguments, or function call parameters slips through unredacted. For a v1 PII guardrail this scope is acceptable but should be called out in the docstring.
- `tests/.../test_peyeeye.py` — 8 tests covering init paths (env, explicit, missing), pre-call (basic, stateless, empty messages), post-call (rehydrate, no-session noop), and typed-error reraising. Solid coverage of the happy paths and the configured failure paths. **Missing**: any test for the `id(data)` fallback path on `_cache_key`, and any test for the `cache.set_cache` (pre) vs `litellm.cache.get_cache` (post) asymmetry — both of which are real bugs.
- `litellm/types/guardrails.py:94` — `SupportedGuardrailIntegrations.PEYEEYE = "peyeeye"`. `LitellmParams` MRO updated at `:774` to include `PeyeeyeGuardrailConfigModel`. Standard registration pattern.
- Vendor name "peyeeye" / `peyeeye.ai` — the PR description should disclose the vendor relationship of the author (the GitHub handle `tim-peyeeye` is consistent but the convention here is to mention it explicitly in the PR body).

## Risk

Medium. The functional surface is well-isolated (opt-in guardrail, no impact unless the user configures it). But the cache-asymmetry between pre- and post-call hooks is a correctness bug that will cause silent PII placeholder leakage in any deployment that hasn't separately configured `litellm.cache`, which is exactly the deployment shape that would adopt a PII guardrail. The `id(data)` fallback compounds it.

## Verdict

**merge-after-nits** — fix the cache-handle asymmetry (use the same DualCache reference end-to-end, or document and assert the precondition that `litellm.cache` is configured), drop the `id(data)` fallback in favour of an explicit error or warning, and add an integration test that exercises the realistic "no global cache configured" path. Code quality is otherwise good.
