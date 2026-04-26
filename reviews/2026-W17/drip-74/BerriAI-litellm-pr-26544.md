---
pr: 26544
repo: BerriAI/litellm
sha: a6be3bdb9dbd614101b7ecdc9cd46b70bf769f13
verdict: request-changes
date: 2026-04-26
---

# BerriAI/litellm #26544 — feat(guardrails): add peyeeye PII redaction & rehydration guardrail

- **Author**: tim-peyeeye
- **Head SHA**: a6be3bdb9dbd614101b7ecdc9cd46b70bf769f13
- **State**: CLOSED (re-opened as #26546 — this review still applies, the underlying code is the same)
- **Apparent size**: +19,622/-252,708 across 2,042 files — the branch is broken; see "Branch hygiene" below
- **Real reviewable surface**: 4 files in `litellm/proxy/guardrails/guardrail_hooks/peyeeye/` plus types and tests

## Scope (intended)

Adds a `peyeeye` guardrail integration with two coordinated hooks:

- `async_pre_call_hook` calls `POST /v1/redact` to swap PII out of every text-bearing message part, caches the returned `session_id` keyed by `litellm_call_id`, and lets the request proceed with placeholders.
- `async_post_call_success_hook` looks up the cached `session_id`, calls `POST /v1/rehydrate` against each text part of the model's response to swap placeholders back to the original PII, then deletes the session both from the peyeeye API (best-effort) and from the local cache.

Two `peyeeye_session_mode`s — `stateful` (server stores the token→value map under a `ses_…` id) and `stateless` (server returns a sealed `skey_…` blob, server-side state retained for nothing). Default is `stateful`.

## Specific findings

### 🔴 Bug: pre-call writes to one cache, post-call reads from a different one

This is the headline issue. In the new file `litellm/proxy/guardrails/guardrail_hooks/peyeeye/peyeeye.py`:

- Pre-call hook (lines ~155-165 of the patch) writes the session id into the `cache: DualCache` parameter passed in by the framework:
  ```python
  cache.set_cache(cache_key, session_id, ttl=SESSION_CACHE_TTL_SECONDS)
  ```
- Post-call hook (lines ~180-185 of the patch) reads from the **module-level** `litellm.cache`:
  ```python
  session_id = litellm.cache.get_cache(cache_key) if litellm.cache else None
  ```

These are two different caches in any non-trivial deployment. `DualCache` (the one threaded into `async_pre_call_hook`) is the proxy's runtime cache instance with both Redis and in-memory backends. `litellm.cache` is the global completion-response cache, which may be `None`, may have a different backend, and on a stock proxy install **is `None`** — meaning `session_id` will be `None`, the post-call hook will return early at the "No redaction happened (or session expired)" branch, and **every model response will reach the user with peyeeye placeholders un-rehydrated.** That's a silent data corruption bug, not a rehydration miss.

The cleanup branches at lines ~210-220 also use `litellm.cache.delete_cache`, so they'll never run for the cached entry that was actually written. The session id remains in `DualCache` until TTL expiry (1 hour).

**Fix**: store the cache reference on `self` in `async_pre_call_hook` (or use the proxy's standard cache accessor). Don't reach for `litellm.cache` in the post-call path.

### 🟡 Race / re-entrancy: `_cache_key` falls back to `id(data)`

`_cache_key` at the bottom of the file:
```python
return f"peyeeye_session:{data.get('litellm_call_id') or id(data)}"
```

`litellm_call_id` is normally set by the time guardrails run, but `id(data)` as fallback is unsafe — Python may reuse object ids after GC, so two unrelated requests could collide. If `litellm_call_id` is missing this should be a hard error or skip (not fall through to a nondeterministic key). At minimum, log a warning so the failure mode is visible in production.

### 🟡 Failure semantics: rehydrate failure silently returns placeholders

In `_rehydrate`:
```python
except Exception as e:
    verbose_proxy_logger.warning("peyeeye: rehydrate failed: %s", e)
    return text
```

So if the peyeeye API is down for the rehydrate path, the user receives the model's response **with `[NAME_001]` style placeholders in it**. That's worse than the failure-loud alternative (raise → 502 to the user). Should be configurable — `on_rehydrate_failure: ["fail_open" | "fail_closed"]` defaulting to `fail_closed` for a security-positioned product.

### 🟡 No call-type gating on multimodal / audio / image

`async_pre_call_hook` accepts call_types including `image_generation`, `moderation`, `audio_transcription`, `embeddings`. The redaction/rehydration logic only operates on text content (`_iter_message_text` skips non-text parts), so pre-call is a no-op for those — but the cache write still happens *if* a session was returned, and post-call will try to swap placeholders in `ModelResponse.choices[].message.content` which is meaningless for image/audio responses. Either `should_run_guardrail` already gates on this (please confirm) or this needs an explicit `call_type in {"completion", "text_completion", "anthropic_messages"}` early-return.

### 🟡 `stateless` cleanup branch is silently skipped

```python
if self.peyeeye_session_mode == "stateful" and session_id.startswith("ses_"):
    try:
        await self._delete_session(session_id)
```

In stateless mode the session id is a sealed `skey_…` blob and there's nothing to delete server-side — that's correct. But the `litellm.cache.delete_cache(cache_key)` call right below it is gated only by the outer `try/except: pass`. Combined with finding #1 above, the cache entry for the in-flight request never gets cleaned up either. After fixing #1, double-check the stateless cleanup path also evicts from the right cache.

### 🟢 Generally good shape

- `_iter_message_text` / `_set_message_text` correctly handle both string and `[{type: "text", text: ...}]` multimodal content shapes.
- Status-code → exception class mapping in `_reraise_api_error` is reasonable: 401 → `MissingSecrets`, 429 → `APIError`, others → generic `APIError`. Could also distinguish 5xx for retry policy, but this is a fine v1.
- Batch redaction (`_redact_batch` calls `/v1/redact` once with all message texts) is the right shape — one round-trip per turn rather than one per message part.
- TTL of 3600s on the cached session is sensible (covers slow first-token streaming, long tool-call loops).

### 🔴 Branch hygiene: 2,042 files, -252k lines

This PR's diff is unreviewable because the branch deleted half of `docs/my-website/` (1063 lines from `anthropic_opus_4_5_and_advanced_features/index.md`, 138 lines from `anthropic_wildcard_model_access_incident/index.md`, etc. etc.) and rewrote configs across `.circleci/`, `.github/workflows/`, Dockerfiles, and 1,800 other files. This is almost certainly the result of branching off a stale base and then merging `main` in a way that produced an inverted diff. The PR was correctly closed for this reason. The replacement (#26546) needs to be re-cut against current `main` with **only** the four peyeeye files plus `pyproject.toml` if a `[tool.litellm.guardrails]` registry hook is needed.

## Risk

If the cache-mismatch bug (finding #1) ships as-is, every redacted request will return placeholder strings to the user. That's a customer-visible regression, plus it leaks the *fact* of redaction (a privacy signal in itself) and makes the integration look broken. Combined with the silent `_rehydrate` failure path, the "happy path of failure" leaks placeholders to users by default.

## Verdict

**request-changes** — the code shape is right and the design (pre-call redact + cache session id + post-call rehydrate + best-effort cleanup) is exactly how this should work. But the cache-mismatch bug between `cache` (DualCache) and `litellm.cache` will silently corrupt every redacted response in production. Plus the branch needs to be re-cut from clean `main` (which the maintainer caught — hence the close).

Required before merge of the re-cut PR:
1. Use the same cache instance in both hooks. Store the `DualCache` reference on `self` in `async_pre_call_hook`, or wire up the proxy's standard guardrail-cache accessor.
2. Make rehydrate-failure behavior configurable, default `fail_closed` (raise → 502) for a privacy-positioned product.
3. Hard-error or warn-loud on missing `litellm_call_id` instead of falling back to `id(data)`.
4. Add a positive end-to-end regression test that round-trips a request with PII through both hooks against a stub peyeeye server and asserts the user sees the original PII *and* the cache is empty after the response.

## What I learned

The `cache` parameter passed into `async_pre_call_hook` versus the module-level `litellm.cache` is a foot-gun this codebase has hit before — they look equivalent but aren't. New guardrail authors keep reaching for `litellm.cache` because it's the obvious global, missing that the framework already handed them the right object. Worth a comment block at the top of the `CustomGuardrail` base class: "the `cache` argument is *not* `litellm.cache`; use it for per-request scratch state."
