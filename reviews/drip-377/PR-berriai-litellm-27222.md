# BerriAI/litellm #27222 — [Feat] Decouple S3 audit-log config via s3_audit_callback_params

- Head SHA: `3a01436c00826b69055bfea871cdafcb5179f42c`
- Diff: +297 / -59 across 7 files

## Findings

1. The core decoupling is well-shaped. `litellm/integrations/s3_v2.py:53-77` adds a `s3_callback_params_override: Optional[dict]` constructor kwarg, and `_init_s3_params` at `:148-228` is refactored to read from a single `params_source` dict (defaulting to `litellm.s3_callback_params or {}` when the override is `None`). The previous implementation directly mutated `litellm.s3_callback_params[key] = litellm.get_secret(value)` to resolve `os.environ/X` markers in-place — this PR correctly switches to a local dict-comprehension at `:163-170` that resolves into a fresh dict and *never mutates the source*. This is the right invariant for supporting two parallel S3 logger instances (regular + audit) without one polluting the other's config.
2. The new module-level `s3_audit_callback_params: Optional[Dict] = None` at `litellm/__init__.py:391` follows the established convention for callback-params globals. Concern: there is no documentation of what config keys are expected in this dict (the PR body says "the same setting" but doesn't enumerate); an inline comment listing supported keys (`s3_bucket_name`, `s3_region_name`, etc.) would prevent the inevitable "I set s3_audit_bucket_name and it didn't work" support burden.
3. The two test files (`tests/test_litellm/integrations/test_s3_v2.py` +71, `tests/test_litellm/proxy/management_helpers/test_audit_log_callbacks.py` +123) provide solid coverage — but the audit-log callback test is heavier than the s3_v2 unit test, which suggests the integration path is the trickier surface. Worth confirming the test exercises the case where `s3_callback_params` and `s3_audit_callback_params` resolve different `os.environ/` markers (the `params_source` separation is the regression target).
4. `audit_logs.py` (+30/-9) and `proxy_server.py` (+20/-0) carry the wiring. The `proxy_server.py` change presumably constructs a second `S3Logger` with `s3_callback_params_override=litellm.s3_audit_callback_params` — visible diff is truncated, but verify the construction is gated on `litellm.s3_audit_callback_params is not None` so the existing single-bucket users are unaffected (no behavioural change when the new global stays at default `None`).

## Verdict

**Verdict:** merge-after-nits
