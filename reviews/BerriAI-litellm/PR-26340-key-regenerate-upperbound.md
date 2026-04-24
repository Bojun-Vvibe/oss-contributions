# PR-26340 — fix(key_management): enforce upperbound_key_generate_params on /key/regenerate

[BerriAI/litellm#26340](https://github.com/BerriAI/litellm/pull/26340)

## Context

Two-line behavior fix in
`litellm/proxy/management_endpoints/key_management_endpoints.py` —
inside `_execute_virtual_key_regeneration`:

```python
if data is not None:
    # Enforce upperbound key params on regenerate (don't fill defaults)
    _enforce_upperbound_key_params(data, fill_defaults=False)
    non_default_values = await prepare_key_update_data(
        data=data, existing_key_row=key_in_db
    )
```

The `_enforce_upperbound_key_params` function already existed and was
applied to `/key/generate` and `/key/update`. It clamps fields like
`duration`, `max_budget`, `models`, `max_parallel_requests`, etc.
against the proxy admin's configured `litellm.upperbound_key_generate
_params`. The `/key/regenerate` path was the only key-mutation
endpoint that bypassed it — meaning a user with a key already at the
configured `max_budget=10.0` could call `/key/regenerate` with
`max_budget=500.0` and walk away with a 50× budget bump. Same for
`duration`. Classic privilege-escalation-via-forgotten-codepath.

`fill_defaults=False` is the important parameter: on regeneration,
fields that the caller didn't pass (`None`) should *inherit from the
existing key*, not be reset to upperbound defaults.

The PR ships four substantial async tests against
`_execute_virtual_key_regeneration` directly:

1. `test_execute_virtual_key_regeneration_rejects_over_limit_duration` —
   upperbound `duration="1h"`, request `duration="2h"` → expects
   `HTTPException(status_code=400, detail contains "duration")`, and
   asserts `mock_prisma_client.db.litellm_verificationtoken.update.
   await_count == 0` (the rejected request never reaches the DB).
2. `test_execute_virtual_key_regeneration_allows_within_limit_duration`
   — request `duration="30m"` under the same upperbound → succeeds,
   `await_count == 1`.
3. `test_execute_virtual_key_regeneration_rejects_over_limit_max_budget`
   — upperbound `max_budget=10.0`, request `max_budget=500.0` →
   rejected with detail containing `"max_budget"`. This is the proof
   that the fix covers non-duration fields.
4. `test_execute_virtual_key_regeneration_skips_none_values` — empty
   `RegenerateKeyRequest()` (all fields None) under the same
   upperbound → succeeds without raising. This locks in the
   `fill_defaults=False` semantic: None means "inherit", not "fail
   validation as if user requested unbounded".

Each test patches the I/O surfaces (`get_new_token`,
`_insert_deprecated_key`, `_delete_cache_key_object`, the rotation
hook) and uses a tiny `DictLikeResult` shim so the
`update`/`create` returns iterate cleanly.

## Why it matters

This is a **server-side-trust-boundary** bug: the regenerate endpoint
was implicitly trusting the caller to not request bumps beyond what
the proxy admin allowed. For multi-tenant proxies (the entire reason
upperbound exists), this is a privilege-escalation primitive — any
authenticated user with regenerate rights on a key could grant
themselves arbitrary budget by passing the desired max_budget through
regenerate instead of update.

## Strengths

- The fix is minimal: import-and-call. No surface change. Same
  function, same parameters as the other endpoints — consistency over
  novelty.
- `fill_defaults=False` is the *correct* mode for regenerate. The
  `True` mode would have caused regression: existing keys whose
  duration/budget pre-dated a tightened upperbound would suddenly
  raise on every regenerate. The `False` mode preserves grandfathered
  values and only clamps *new* requests.
- The test for `None`-only fields is the one that tells me the author
  understood the semantic. Without it, a future maintainer "tightening"
  the validator could flip the default to `fill_defaults=True` and
  break every existing key on rotation.
- Asserting `update.await_count == 0` on rejection is the correct
  shape — proves the rejection short-circuits *before* any DB write.
  Many privilege-escalation fixes ship with "raises HTTPException"
  asserts but no proof the side effect was suppressed.
- The `max_budget` test specifically guards the worst-case impact
  (financial), proving the fix isn't just a duration rate-limiter.
- The four tests cover the 2x2 matrix: (over-limit | within-limit) ×
  (duration | max_budget | None) — minus the within-limit max_budget
  case which would be redundant.

## Concerns / risks

- `_enforce_upperbound_key_params(data, fill_defaults=False)` mutates
  `data` in place (based on the surrounding code's behavior). If
  `data` is shared with a caller above (it's a Pydantic
  `RegenerateKeyRequest`), in-place mutation can leak. The test setup
  builds a fresh `data` per test so doesn't catch this; not a blocker
  but worth confirming.
- No test exercises the interaction with `models` or
  `max_parallel_requests` upperbounds. The PR title and code comment
  imply general enforcement, but only `duration` and `max_budget` are
  proven by tests.
- `_enforce_upperbound_key_params` raises `HTTPException` directly.
  The endpoint wrapper turns that into a 400; if any caller of
  `_execute_virtual_key_regeneration` is internal (non-HTTP),
  surfacing an HTTPException up the stack is awkward. The test
  imports `HTTPException` from the function-under-test's expected
  exception, so the contract is at least pinned down.
- Rate-limit semantics aren't tested: rapid regenerates from the same
  key shouldn't be a way to repeatedly try to push past upperbound.
  Probably handled at a different layer, but the comment in the diff
  doesn't mention it.
- The mock prisma client (`_make_regenerate_mock_prisma`) doesn't
  exercise the deprecated-key insert path or the rotation hook. Those
  are patched-away. Coverage of the success path is therefore narrower
  than it looks.

## Suggested follow-ups

- Add a regression test asserting *audit log* row is *not* written on
  rejection — silent rejects without audit trace would be bad for
  forensic review.
- Add tests for `models` and `max_parallel_requests` enforcement on
  regenerate, even if just smoke tests, to nail down the "general
  enforcement" claim.
- Consider whether `_execute_virtual_key_regeneration` should
  short-circuit *earlier* — i.e. before any I/O like
  `get_new_token` — so a rejected regenerate doesn't burn an entropy
  draw. Looking at the diff, the enforcement is placed before
  `prepare_key_update_data` but it's worth checking that it's also
  before any DB read.
- Document the `fill_defaults` flag's semantics in
  `_enforce_upperbound_key_params` itself (`True` for create-style
  paths, `False` for update/regenerate paths).
- Mirror this audit across other "back door" endpoints — e.g. team-key
  regenerate, organization-key endpoints — anywhere a key-mutating
  path could plausibly bypass `_enforce_upperbound_key_params`.
