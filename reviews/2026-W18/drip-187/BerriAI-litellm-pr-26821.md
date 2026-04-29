---
pr: BerriAI/litellm#26821
sha: 2ccb4b94e548edb1612935884abf6b09a283ab1e
verdict: merge-as-is
reviewed_at: 2026-04-30T00:00:00Z
---

# fix(proxy/auth): tighten guardrail modification permission check

URL: https://github.com/BerriAI/litellm/pull/26821
Files: `litellm/proxy/auth/auth_checks.py`, `tests/test_litellm/proxy/auth/test_auth_checks.py`

## Context

`_user_requested_modification` at `auth_checks.py:374` decides whether the
request's metadata block expressed *intent to modify* the guardrail config,
which gates the subsequent `can_modify_guardrails` permission check. The
prior implementation was:

```python
return any(coerced.get(key) for key in _GUARDRAIL_MODIFICATION_KEYS)
```

This is truthiness-based: an explicitly-supplied falsy value
(`{}`, `[]`, `""`, `0`, `False`) returned `False`, so the auth check was
skipped — but downstream guardrail evaluation still inspected the *presence*
of those keys. A request with `metadata={"guardrails": {}}` was therefore
parsed by the guardrail layer as "disable all guardrails" while the auth
layer silently let it through with no permission check. That is an auth
bypass — privileged behavior unlocked by a key whose value happened to be
falsy.

## What changed

One-line fix at `auth_checks.py:377`:

```python
return any(key in coerced for key in _GUARDRAIL_MODIFICATION_KEYS)
```

Now any *presence* of one of `_GUARDRAIL_MODIFICATION_KEYS` triggers the
permission check, regardless of value truthiness. The semantic now matches
what the downstream guardrail evaluator already does (presence-based), so
auth and evaluation agree on what "the user is trying to modify guardrails"
means.

The regression test at `test_auth_checks.py:2081-2106` is the right shape:
parametrize over all four keys × five falsy values (20 cases), patch
`can_modify_guardrails` to return `False`, assert HTTP 403. This locks in
the contract that "metadata key present" = "permission required" for every
falsy value the prior bug let through.

## Risks

- Behavior change for callers that legitimately sent an empty `guardrails`
  dict expecting no-op semantics — they now hit the permission check.
  This is the *correct* behavior (presence implies intent to override the
  default) and matches the downstream evaluator, but ops should keep an
  eye on any 403 spike from internal tooling that was relying on the old
  silent-passthrough.

## Verdict

`merge-as-is` — minimal, correct security fix, parametrized test covers
every falsy variant of every key, and the new semantics finally align the
auth gate with downstream evaluation. Exactly the kind of one-line auth
fix that should land fast.
