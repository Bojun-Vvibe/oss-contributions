# PR #26617 — fix: DB model 'litellm_params' not applied to requests

- **Repo**: BerriAI/litellm
- **PR**: #26617
- **Head SHA**: `11f6a1c7`
- **Author**: bringCool
- **Size**: +147 / -0 across 2 files
- **URL**: https://github.com/BerriAI/litellm/pull/26617
- **Verdict**: **merge-after-nits**

## Summary

Closes a real router-side bug where deployment-level `litellm_params`
fields configured via the DB (dict shape) or via Pydantic
`LiteLLM_Params` (code shape) were *parsed and stored* but not
*forwarded* into the per-request kwargs that actually reach the LLM
client. The narrow case driving the fix is `use_chat_completions_api`
— a deployment-level flag that toggles a model from the Responses API
shape to the Chat Completions API shape — but the implementation is a
small explicit allowlist that can be extended as more deployment-level
params need forwarding. Fix is the right shape: a 12-line additive
block in `_update_kwargs_with_deployment` with a hard-coded allowlist
of forwardable keys (currently just `("use_chat_completions_api",)`)
and `kwargs.setdefault(...)` semantics so request-level overrides win.

## Specific changes

- `litellm/router.py:2438-2447` — new block at the end of
  `_update_kwargs_with_deployment`:
  ```python
  _deployment_litellm_params = deployment.get("litellm_params", {}) or {}
  if not isinstance(_deployment_litellm_params, dict):
      _deployment_litellm_params = _deployment_litellm_params.model_dump(
          exclude_none=True
      )
  for _key in ("use_chat_completions_api",):
      _value = _deployment_litellm_params.get(_key)
      if _value is not None:
          kwargs.setdefault(_key, _value)
  ```
  The shape correctly handles both branches:
  (a) `deployment["litellm_params"]` is a dict (the DB-loaded path
  via `_create_deployment` from a model_list entry), and
  (b) it's a Pydantic `LiteLLM_Params` (the code-configured path) —
  the latter is normalized to dict via `.model_dump(exclude_none=True)`.

## Tests

Four test cases at `tests/test_litellm/test_router.py:2820-2956` —
all named explicitly:

- `test_update_kwargs_with_deployment_forwards_use_chat_completions_api_dict`
  (`:2820`): builds a `Router`, hand-builds a deployment dict with
  `litellm_params.use_chat_completions_api: True`, calls
  `_update_kwargs_with_deployment`, asserts `kwargs["use_chat_completions_api"] is True`.
- `test_update_kwargs_with_deployment_forwards_use_chat_completions_api_pydantic`
  (`:2854`): same but goes through the model_list path which produces
  a Pydantic-shaped `litellm_params`. Includes the sanity assertion
  `assert not isinstance(deployment.litellm_params, dict)` to confirm
  the test is exercising the *Pydantic* branch — a great touch
  because without it a future refactor that flattens to dict would
  silently turn this into a duplicate of the previous test.
- `test_update_kwargs_with_deployment_request_overrides_use_chat_completions_api`
  (`:2884`): deployment says `True`, request kwargs say `False`,
  asserts the request kwarg wins. This pins the `setdefault`
  semantics and is the regression guard against a future "fix" that
  changes `setdefault` to `kwargs[...] = value` (which would silently
  override per-request overrides — exactly the kind of bug that
  bites in production three months later when someone tries to
  override a per-deployment default for a single call).
- `test_update_kwargs_with_deployment_does_not_forward_router_internal_params`
  (`:2917`): hand-builds a deployment with `tpm`, `rpm`, `max_budget`,
  `litellm_credential_name` on `litellm_params`, asserts none of those
  leak into request kwargs. This is the critical "allowlist not
  blocklist" regression guard — without it, the next person to
  generalize the loop ("just forward all `litellm_params` keys") would
  silently leak router-internal params into the LLM call, which would
  either be ignored (best case) or rejected by the provider with a
  400 (worst case). The test names the four most dangerous keys
  explicitly and asserts each is absent.

## Risks

- **Allowlist size = 1 today**. The current `("use_chat_completions_api",)`
  tuple covers exactly the bug the PR title names. As more
  deployment-level forwardable params get added, the tuple grows —
  with the existing test pattern, every addition needs paired tests
  (forward-dict, forward-pydantic, request-overrides, internal-params-not-leaked).
  Worth a `# IMPORTANT: When adding a key, update the four
  test_update_kwargs_with_deployment_* tests to cover both shapes
  and both override directions.` comment at the tuple definition so
  the next contributor doesn't silently skip the matrix.
- **`exclude_none=True` for the Pydantic branch**: this is the right
  call (a Pydantic `None` shouldn't override a request-level value
  via `setdefault`) but it has a subtle interaction — if a deployment
  *explicitly* sets `use_chat_completions_api: None` (intent: "fall
  back to request-level"), the dict branch and Pydantic branch behave
  the same way (no forwarding), which is what you want. But if Pydantic
  ever surfaces a default-non-None value for a forwardable key, the
  `model_dump(exclude_none=True)` semantics still preserve it, so
  it's safe.
- **No coverage for absence-on-deployment, presence-on-request**:
  i.e. deployment doesn't set `use_chat_completions_api` at all,
  request kwargs do. Should be a no-op (the for loop's
  `_value is not None` guard skips the `setdefault`), but worth a
  fifth test pinning that explicitly. The current four tests don't
  cover the "deployment is silent" path.

## Verdict

**merge-after-nits**: the fix is correctly shaped (allowlist not
blocklist, `setdefault` not assignment, normalize-then-iterate), the
four tests cover the key matrix, the dual-shape handling (dict +
Pydantic) is clean. Three nits: (1) add a comment at the tuple
definition warning about the test matrix expansion requirement;
(2) add a fifth test pinning "deployment silent → no forwarding";
(3) consider whether other obviously-forwardable
deployment-level fields (e.g. `drop_params`, `additional_drop_params`,
provider-shape-toggle flags besides `use_chat_completions_api`) should
ship in the same PR or as follow-ups — the bug class extends beyond
the one named param.

## What I learned

The "allowlist not blocklist" pattern for kwargs forwarding is the
right default whenever you're moving values from one untrusted/
permissive container (deployment config) into another more
restrictive one (LLM request kwargs). The blocklist alternative
("forward all except [internal keys]") fails open: forget to add
a new internal key to the blocklist and it silently leaks into the
LLM call. The allowlist fails closed: forget to add a new
forwardable key and the bug shape is "feature didn't take effect"
which is a loud user-visible bug that surfaces fast. The
test-matrix shape — dict + Pydantic shapes × forward + override
directions — is the canonical four-test pattern for any
configuration-into-request forwarding code. The
`assert not isinstance(deployment.litellm_params, dict)` sanity
line in the Pydantic test is a particularly good defensive pattern;
it catches the "we accidentally normalized to dict early" refactor
that would otherwise silently turn the Pydantic test into a
duplicate of the dict test without anyone noticing.
