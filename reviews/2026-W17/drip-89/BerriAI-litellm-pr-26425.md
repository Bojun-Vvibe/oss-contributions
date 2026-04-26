---
pr: 26425
repo: BerriAI/litellm
sha: 93df7200870eb053a08c99f7a5fef3608194087d
verdict: merge-after-nits
date: 2026-04-27
---

# BerriAI/litellm #26425 — fix(proxy): downgrade 401/403 auth denials from ERROR to WARNING in auth_exception_handler

- **Head SHA**: `93df7200870eb053a08c99f7a5fef3608194087d`
- **Size**: +27 / −7 plus 2 unrelated formatting-only diffs

## Summary
Splits the single `verbose_proxy_logger.exception(...)` call in `_handle_authentication_error()` into two branches:
- **Expected auth failures** (BudgetExceededError, ProxyException with code 401/403, HTTPException with status 401/403) → `verbose_proxy_logger.warning(...)` *without* a stack trace.
- **Everything else** → keeps the original `verbose_proxy_logger.exception(...)` with traceback at ERROR.

Reduces routine access-control noise that was hitting Datadog/OTEL error budgets and on-call alerts.

## Specific findings
- `litellm/proxy/auth/auth_exception_handler.py:78-93` — the `_is_expected_auth_failure` predicate uses three checks chained with `or`:
  1. `isinstance(e, litellm.BudgetExceededError)` — correct, this is a known business-logic deny.
  2. `isinstance(e, ProxyException) and str(e.code) in ("401", "403")` — `e.code` is coerced via `str()` because `ProxyException.code` can be `int` or `str` depending on the construction site. Defensive; correct.
  3. `isinstance(e, HTTPException) and e.status_code in (401, 403)` — `HTTPException.status_code` is always `int`; the comparison set is `(401, 403)` not `("401", "403")`. Correct.
- `:84-90` — the WARNING-branch message changes from "Exception occured" to "Auth denied", which is more precise and easier to grep for. Good.
- The `extra={"requester_ip": requester_ip}` payload is preserved on both branches. Correct — IP attribution still flows to structured logging sinks.
- **Concern**: 401 and 403 from a *server-side* misconfiguration (e.g. broken JWT validator, expired secret) will now be silently downgraded to WARNING. That defeats the original error-budget signal for the operator. Consider adding a sub-classification: only downgrade `ProxyException` codes that originate from `user_api_key_auth` (key not found, budget exceeded, model not allowed) and keep the `exception()` path for `HTTPException(401)` raised from JWT-decode or similar plumbing failures.
- The two formatting-only diffs (`litellm_core_utils/prompt_templates/factory.py:5294` and `llms/predibase/chat/transformation.py:178+208+350`) are noise from a pre-commit autoformatter. They should be in a separate "chore: black/ruff format" PR; their presence here makes the security-relevant diff harder to bisect. Same complaint applies to the `xecguard.py:215-220` reformat and the trailing-newline addition at `xecguard.py:591`.
- Test coverage: PR body doesn't mention a unit test asserting 401-becomes-WARNING / 500-stays-ERROR. The branch logic is small but ill-tested would silently regress on a future refactor. A two-test addition with a fake logger + mock exceptions would cover the four cases (BudgetExceeded/ProxyException401/ProxyException500/HTTPException403) in ~30 lines.

## Risk
Medium-low. The downgrade is correct for the *common* expected case but loses an alerting signal for the JWT-broken corner case, and the unrelated formatting diffs increase merge-conflict surface.

## Verdict
**merge-after-nits** — split out the 4 unrelated formatting diffs into a separate chore PR; add a regression test for the four exception-class branches; consider preserving `exception()` for `HTTPException(401/403)` raised outside the `user_api_key_auth` codepath.
