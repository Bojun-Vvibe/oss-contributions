# BerriAI/litellm #26714 — Security: gate skills auto-execution and guardrail exec() behind env vars

- PR: https://github.com/BerriAI/litellm/pull/26714
- Head SHA: `67fb5adbfd28cf33f7f00d3421a1e93f10ba08a7`
- Files: `litellm/proxy/guardrails/guardrail_endpoints.py`, `litellm/proxy/guardrails/guardrail_hooks/custom_code/custom_code_guardrail.py`, `litellm/proxy/hooks/__init__.py`, `tests/test_litellm/proxy/guardrails/guardrail_hooks/test_response_rejection_guardrail_code.py`, `tests/test_litellm/proxy/guardrails/test_custom_code_security.py`

## Citations

- `litellm/proxy/hooks/__init__.py:27` — `"litellm_skills": SkillsInjectionHook` removed from the unconditional `PROXY_HOOKS` dict.
- `litellm/proxy/hooks/__init__.py:35-36` — gated re-add: `if os.getenv("LITELLM_ENABLE_SKILLS", "false").lower() == "true": PROXY_HOOKS["litellm_skills"] = SkillsInjectionHook`.
- `custom_code_guardrail.py:147-153` — new static method `_require_custom_code_enabled()` raises `CustomCodeCompilationError("disabled by default. Set LITELLM_ENABLE_CUSTOM_CODE_GUARDRAILS=true to enable.")` when the env var isn't `"true"`.
- `custom_code_guardrail.py:156` — `_do_compile()` calls `self._require_custom_code_enabled()` *before* `compile_sandboxed(self.custom_code)` and the subsequent `exec(compiled, exec_globals)`. Gate is at the only execution chokepoint.
- `guardrail_endpoints.py:2073-2080` — `test_custom_code_guardrail` admin endpoint also checks the env var and returns a structured failure (`success=False, error_type="compilation"`) instead of raising; preserves existing response shape so callers don't need to change.
- Tests: both `test_response_rejection_guardrail_code.py` and `test_custom_code_security.py` get an `autouse=True` `enable_custom_code_guardrails` fixture that monkeypatches the env var to `"true"`, keeping the existing red-team byte-code-rewrite tests runnable.

## Verdict

`merge-as-is`

## Reasoning

This is exactly the security gate this code needs. There are two latent foot-guns and the fix is the same shape for both: opt-in via explicit env var, deny by default, no behavior change for users who don't know about the feature.

**Foot-gun 1: SkillsInjectionHook.** Loaded by default in every proxy boot via the `PROXY_HOOKS` dict. Skills are markdown files with frontmatter that get *injected into the system prompt* — meaning a deploy that happens to ship with skills directories from a default location (or worse, a misconfigured shared skills dir) silently rewrites the system prompt for every request. That is a remote-instruction-injection surface even when the skills source is "internal", because anyone with write access to the skills dir gains write access to the system prompt. Gating behind `LITELLM_ENABLE_SKILLS=true` makes the operator opt in deliberately and removes the surface for everyone else.

**Foot-gun 2: CustomCodeGuardrail.** This is the more serious one. The guardrail compiles user-supplied Python and `exec()`s it against `build_sandbox_globals()`. The companion test `test_custom_code_security.py` already documents one bytecode-rewrite escape that abuses `code.replace(co_names=...)` to swap a function's bytecode and read the real builtins dict — i.e. the sandbox is documented-breakable. Defense-in-depth says don't run untrusted Python through `exec()` *at all*, and "by default the feature is off, you have to flip a switch to use it" is the correct posture for a feature whose own test file enumerates known sandbox escapes. The gate landing inside `_do_compile()` (`custom_code_guardrail.py:156`) is the right placement — it's the only chokepoint, called from the lock-acquired path, so concurrent compile attempts all hit the same gate without TOCTOU. The mirror gate in the admin `test_custom_code_guardrail` endpoint (`guardrail_endpoints.py:2073`) is the right secondary placement: even an admin shouldn't be able to indirectly run untrusted code via the test endpoint when the feature is off.

The error shape is well-chosen. `CustomCodeCompilationError` is the existing error type for this code path, so callers' error-handling already knows how to surface it; the new gate doesn't introduce a new exception type the callers haven't seen. The admin endpoint returns `success=False, error_type="compilation"` which keeps the response shape stable — clients that already handle compilation failures keep working without any code change.

The test fixtures (`autouse=True` env-var monkeypatch) are the right shape. Not just because they keep the existing red-team tests runnable, but because they document at the fixture level *the contract*: "to test custom code, you must enable it." Future tests will pick this up automatically and any test that *doesn't* want the gate flipped can `monkeypatch.delenv` locally.

Two posture observations (not blocking):

- The env-var check `os.getenv(...).lower() != "true"` is permissive of `"True"`, `"TRUE"`, `"true "` (no, the trailing space would fail `.lower() != "true"` correctly — but `"True "` with trailing space *would* fail, which is fine: strict matching is the right posture for a security gate).
- Consider logging at startup which gates are active (a single line: `[litellm] custom_code_guardrails=disabled, skills=disabled` or similar). Operators reading boot logs would then know they're protected without grepping config.

Ship.
