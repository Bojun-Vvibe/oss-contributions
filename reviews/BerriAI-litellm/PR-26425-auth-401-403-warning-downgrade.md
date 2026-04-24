# litellm PR #26425 — fix(proxy): downgrade 401/403 auth denials from ERROR to WARNING in auth_exception_handler

Link: https://github.com/BerriAI/litellm/pull/26425

## What it changes

`auth_exception_handler.py` previously called `logger.exception(...)`
unconditionally, so every routine "wrong key" / "model not in ACL" /
"budget exceeded" denial wrote a full traceback at ERROR level. In a
multi-tenant proxy, the noise floor was high enough to bury actual
internal errors. This PR distinguishes:

- `ProxyException` / `HTTPException` with status 401 or 403 →
  `logger.warning(...)`, no traceback.
- Anything else → `logger.exception(...)` (unchanged).

The PR description shows a clean before/after log diff. The functional
diff in `auth_exception_handler.py` is small (~16 net lines); the
remainder of the +8k/-853 diff is incidental re-export of a large
batch of unrelated provider/transformation/test changes that look
like a stale rebase.

## Critique

The behavior change is correct: 401/403 from `auth_checks` are
*expected* events on a multi-tenant proxy, not errors, and structured
log levels exist precisely so operators can filter them out. WARNING
+ no-traceback is the right shape — operators can still grep for
denials, but the SRE pager is not woken up by a typo'd API key.

Concerns:

1. **Diff hygiene.** ~8k added / ~853 deleted across 80+ files for
   a logging-level change is a red flag. Files like
   `tests/pass_through_tests/package-lock.json` (+3930),
   `litellm/llms/dashscope/image_generation/transformation.py`
   (+204 brand-new file), and the Anthropic experimental
   pass-through reasoning code are clearly unrelated. Either the
   PR author rebased onto a stale main and accidentally swept up
   half a sprint, or this PR is bundling unrelated work. As-is,
   reviewing the actual claimed change requires manually filtering
   `auth_exception_handler.py` out of the noise. Should be split.
2. **Predicate scope.** `ProxyException OR HTTPException` is the
   right *necessary* condition, but the predicate also needs to
   confirm the exception originated from the auth pipeline. If a
   downstream provider call raises `HTTPException(401)` (e.g., the
   upstream LLM rejected the proxy's own credentials), we now log
   that as a routine auth denial when it is actually an outage
   signal. Worth gating on call-site or exception subclass.
3. **WARNING-without-traceback loses correlation data.** Even for
   an expected 401, having the exception's `details` /
   `proxy_exception_type` in the log line is useful for
   "why was this user denied?" debugging. The before/after example
   shows the WARNING line carries `Auth denied - ...` — verify
   the `...` includes the same denial-reason string that the
   traceback used to provide.

## Suggestions

- Split this PR: keep only `auth_exception_handler.py` + its
  targeted test; rebase the rest onto a separate branch.
- Tighten the predicate so only auth-pipeline-originated 401/403
  are downgraded; upstream-provider 401s should remain ERROR.
- Confirm the WARNING line preserves the `proxy_exception_type` /
  denial-reason fields for downstream log analytics.

Verdict: right intent, wrong PR shape. Re-submit as a focused diff.
