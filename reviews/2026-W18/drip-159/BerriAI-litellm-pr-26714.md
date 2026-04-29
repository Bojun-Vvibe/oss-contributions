# Review: BerriAI/litellm#26714 — Security: gate skills auto-execution and guardrail exec() behind env vars

- **Author**: johnpippett
- **Head SHA**: `67fb5adbfd28cf33f7f00d3421a1e93f10ba08a7`
- **Diff**: +33 / −1 across 5 files

## What it does

Closes two security-class default-on behaviors in the proxy:

1. **Skills auto-execution** — `litellm/proxy/hooks/__init__.py:23-37` removes `"litellm_skills": SkillsInjectionHook` from the unconditional `PROXY_HOOKS` dict, then re-adds it inside an `if os.getenv("LITELLM_ENABLE_SKILLS", "false").lower() == "true":` block alongside the existing `LEGACY_MULTI_INSTANCE_RATE_LIMITING` env-gated entry. Pre-PR every proxy startup loaded `SkillsInjectionHook`, which (per its own contract) injects skill content into request flows automatically; post-PR an operator must opt in.

2. **Guardrail `exec()` payload** — `litellm/proxy/guardrails/guardrail_hooks/custom_code/custom_code_guardrail.py:147-156` adds a static method `_require_custom_code_enabled()` that raises `CustomCodeCompilationError("Custom code guardrails are disabled by default. Set LITELLM_ENABLE_CUSTOM_CODE_GUARDRAILS=true to enable.")` when the env var isn't `true`, and calls it as the first line of `_do_compile`. The endpoint side (`guardrail_endpoints.py:2074-2080`) mirrors the gate at the `test_custom_code_guardrail` admin endpoint, returning a `TestCustomCodeGuardrailResponse(success=False, error=..., error_type="compilation")` instead of executing.

3. **Test fixtures** — `tests/test_litellm/proxy/guardrails/guardrail_hooks/test_response_rejection_guardrail_code.py:9-13` and `tests/test_litellm/proxy/guardrails/test_custom_code_security.py:11-15` both add an `@pytest.fixture(autouse=True) def enable_custom_code_guardrails(monkeypatch): monkeypatch.setenv("LITELLM_ENABLE_CUSTOM_CODE_GUARDRAILS", "true")` so existing tests still exercise the compile path.

## Concerns

1. **Default-disable for `SkillsInjectionHook` is a behavior change for existing operators.** Anyone relying on skills today will silently lose them on next deploy with no migration log. The PR doesn't emit a startup `verbose_proxy_logger.warning("skills hook now requires LITELLM_ENABLE_SKILLS=true...")` and there's no CHANGELOG entry in the diff. For a security-class default-flip this is the right direction, but operators need a loud signal — at minimum a one-time WARN at proxy startup if a config file references skills features but the env var is unset.

2. **`_require_custom_code_enabled()` raises `CustomCodeCompilationError`, which is the same error class used for actual code-compilation failures.** That collapses two very different operational signals (legitimate compile error vs. policy disable) into the same exception type. Catchers downstream now can't distinguish "user wrote bad code" from "admin disabled the feature." A subclass like `CustomCodeDisabledError(CustomCodeCompilationError)` or a separate `CustomCodeDisabledByPolicy` exception would let observability differentiate without breaking existing handlers.

3. **The endpoint-side gate at `guardrail_endpoints.py:2074-2080` returns 200 with `success=False`, but the `_do_compile` path raises an exception.** Two paths to the same disable signal returning two different shapes: the endpoint says "200 OK, success=False, error_type=compilation"; the hook raises. If a CLI/SDK consumer of the endpoint uses HTTP status to discriminate, it'll never see this disable as a non-200. That's actually consistent with how compilation errors are reported by this endpoint today (200 with `success=False`), so it's defensible — but the symmetry should be called out in a comment so future maintainers don't "fix" it.

4. **Env var naming inconsistency.** `LEGACY_MULTI_INSTANCE_RATE_LIMITING` (no `LITELLM_` prefix), `LITELLM_ENABLE_SKILLS`, `LITELLM_ENABLE_CUSTOM_CODE_GUARDRAILS`. The new vars are correctly prefixed; the legacy one should probably get an alias. Out of scope for this PR but worth a follow-up issue.

5. **No CI claim or testing checklist filled in the PR body.** The body says "Related tests updated and passing" but doesn't link to a CI run. The two changed test files do appear to set the autouse fixture correctly, but there's no test for the *disabled* path (e.g., `test_custom_code_guardrail_disabled_by_default_returns_error`). Without that, a future revert to default-on would only break in production.

6. **`litellm/proxy/hooks/__init__.py` still imports `SkillsInjectionHook` unconditionally** at module load time (the import is at the top of the file, not gated). Cheap, but if `SkillsInjectionHook` ever has expensive import-time side effects (it shouldn't, but), the gate is incomplete. Better: `if os.getenv(...) == "true": from .skills import SkillsInjectionHook; PROXY_HOOKS["litellm_skills"] = SkillsInjectionHook`.

7. **No test for the skills hook gate itself.** The new `if` block in `hooks/__init__.py` has no assertion that, with the env var set, `PROXY_HOOKS["litellm_skills"] is SkillsInjectionHook`, and with it unset, the key is absent. Should be a 6-line `monkeypatch`-driven import-reload test.

## Verdict

**merge-after-nits** — correct security-default-flip for both classes (skills auto-injection and arbitrary-code `exec()` in guardrails), with the right env-var allowlist pattern. Land after (a) CI is green and linked, (b) the disabled-by-default test path is covered for both gates, (c) `CustomCodeCompilationError`-vs-`CustomCodeDisabledError` distinction is decided, and (d) a CHANGELOG / startup-warning is added so operators using `SkillsInjectionHook` today learn about the flip before they ship.

