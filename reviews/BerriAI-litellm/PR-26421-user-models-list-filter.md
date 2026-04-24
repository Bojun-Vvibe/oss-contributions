# PR #26421 — fix(proxy): apply user.models restriction to GET /v1/models list

**Repo:** BerriAI/litellm  
**Surface:** `litellm/proxy/auth/model_checks.py` (new
`get_user_models()`); `litellm/proxy/utils.py`
(`get_available_models_for_user()`); test suite.  
**Severity:** Medium (visibility/ACL parity bug; correct fix shape)

## What changed

`GET /v1/models` correctly intersected the listing with `key.models` and
`team.models` but never consulted `user.models`. Result: a user scoped
to (e.g.) `models=["common-models"]` saw the *full* proxy catalog from
the listing endpoint, but got `401` when actually invoking a model
outside the allowed set. Listing/invocation parity was broken.

The PR adds `get_user_models()` mirroring the existing
`get_key_models()` / `get_team_models()` shape:
- `[]` → returns `[]` (caller treats this as "unrestricted; skip")
- `["all-proxy-models"]` → returns the full proxy list
- `["no-default-models"]` → kept as a sentinel that naturally fails the
  later intersection
- Access-group names → expanded via `_get_models_from_access_groups`
- `dict.fromkeys(...)` dedup preserving order

It is then layered into `get_available_models_for_user()` as an
additional intersection on top of `all_models`.

## What's risky / wrong / missing

1. **The "empty means unrestricted" convention is load-bearing and
   under-asserted.** The new intersection block runs only inside an
   `if user_object and user_object.models:` guard, so an empty list
   correctly bypasses filtering. But `get_user_models([])` *also*
   returns `[]`, so a careless caller invoking it directly and naively
   intersecting with `all_models` would silently filter everything out.
   Recommend adding an explicit comment or, better, a sentinel return
   (`None` for "unrestricted") so the contract can't be misread.

2. **Cache freshness.** `get_user_object()` reads through
   `user_api_key_cache`. If an admin updates `user.models` in the DB,
   this list endpoint will continue to serve the cached old set until
   TTL/eviction. The endpoint that *invokes* a model also goes through
   the same cache, so they'll desync the same way they were already
   desynced before this PR — but worth a note in the PR body. Bonus:
   add a doc line on the expected propagation delay.

3. **Order of operations vs. team/key intersection.** The new filter is
   applied *after* `get_complete_model_list(...)` returns. That's fine
   for visibility, but it is also a separate intersection than what the
   invocation path uses. Confirm by inspection that the invocation path
   (`can_user_call_model` or equivalent) applies these filters with
   identical access-group expansion semantics; otherwise a third parity
   gap will reappear.

4. **`get_user_models` does not handle wildcard model patterns** (e.g.
   `openai/*`, `*-haiku`). The doc says "Returns expanded model list,"
   but `_get_models_from_access_groups` is the only expansion step. If
   `user.models` ever supports BYOK wildcards (cf. existing
   `test_get_complete_model_list_byok_wildcard_expansion`), this path
   silently won't honor them. A defensive `# TODO: wildcard support`
   would help.

5. **Imports inside the function** (`get_user_object`, `get_user_models`)
   are placed inside the conditional branch. That's fine for avoiding
   circular imports, but it means the import cost is paid on every
   listing call. If `get_user_object` is already imported elsewhere in
   `utils.py`, hoist it.

6. **Test gap: no end-to-end test of `/v1/models` itself.** All five new
   tests are unit tests of `get_user_models()`. The actual bug is in
   `get_available_models_for_user()` integration. A small
   `tests/proxy/test_models_endpoint.py` style integration covering
   "user with `models=[common-models]` calls `/v1/models` and only sees
   the group's models" is the test the PR description claims this fix
   delivers.

## Suggested fix

- Change `get_user_models([]) -> []` to return `None` (or a typed
  sentinel) and update callers accordingly, so "unrestricted" is
  unambiguous in the type system.
- Add an integration test through the actual route handler.
- Document the cache-propagation behavior on `user.models` changes.

## Severity

Medium. Information-disclosure-adjacent (a restricted user sees the
catalog of models they can't call) — not a privilege escalation, but a
real ACL parity bug that violates least-surprise. Fix shape is right.
