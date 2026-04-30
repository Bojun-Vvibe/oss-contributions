---
pr-url: https://github.com/BerriAI/litellm/pull/26821
sha: b1b98c3fc7f3
verdict: merge-as-is
---

# fix(proxy/auth): tighten guardrail modification permission check

One-character-class semantic fix at `litellm/proxy/auth/auth_checks.py:377`: changes `return any(coerced.get(key) for key in _GUARDRAIL_MODIFICATION_KEYS)` to `return any(key in coerced for key in _GUARDRAIL_MODIFICATION_KEYS)`. The bug: `_user_requested_modification` was using truthiness of the *value* under each guardrail-modification key (`coerced.get(key)`) to decide whether the caller intended to modify guardrails, which means an explicitly-supplied empty/falsy value (`{"guardrails": {}}`, `{"disable_global_guardrails": []}`, `{"opted_out_global_guardrails": ""}`, `{"guardrails": False}`, `{"guardrails": 0}`) returned `False` from the truthiness test and skipped the permission check. The downstream guardrail-evaluation code, however, *did* interpret `{"guardrails": {}}` and friends as "the caller specified an empty allow-list, therefore disable all guardrails" — the textbook auth/eval-layer disagreement-on-falsy-semantics bug where one layer treats empty-dict as "no input" and the other treats it as "explicit empty input."

The fix replaces value-truthiness with key-presence, which matches the downstream eval layer's semantics exactly: if the caller put the key in the metadata dict, they intended to modify, regardless of value. Paired with a 28-line `@pytest.mark.parametrize` test at `test_auth_checks.py:2081-2107` cross-producting `{"guardrails", "disable_global_guardrails", "disable_global_guardrail", "opted_out_global_guardrails"} × {{}, [], "", 0, False}` (20 cases total) and asserting each one 403s when `can_modify_guardrails` is patched to `False`. The test's docstring explicitly names the bug shape (`Truthiness-based gating let callers bypass the check by sending e.g. metadata={"guardrails": {}}, which downstream evaluation interpreted as "disable all guardrails" while the auth layer treated it as no-op`) which is exactly the comment future readers need to not re-introduce the truthiness check.

The choice of `key in coerced` over `coerced.get(key, _SENTINEL) is not _SENTINEL` is the right one — it's the idiomatic Python and works for any `Mapping`-shaped `coerced`. The earlier `_coerce_to_dict` helper (called at `:374`) is what guarantees `coerced` is a real dict at the point of the `in` test, so the membership check is well-defined.

The 4×5 parametrize matrix is the right test-coverage shape for this class of bug: it pins not just "the fix works for `{}`" but also "the fix works for `[]`, `""`, `0`, `False`, and the `disable_global_guardrail` singular variant alongside the `disable_global_guardrails` plural variant" — the latter being the kind of typo that's easy to introduce in a future "refactor the modification-key set" PR.

No nits. The fix is one-line, the test is exhaustive in the dimensions that matter, the docstring documents the bug shape so the contract survives future refactors, and there's no behavioral change for callers that *did* specify a non-falsy value (the `key in coerced` test is `True` whenever `coerced.get(key)` was truthy).

## what I learned
Auth-layer-vs-eval-layer disagreement on what "empty" means is one of the most common authorization-bypass classes in proxy codebases — auth says "the caller didn't specify anything, allow," eval says "the caller specified empty, disable everything," and the gap between those two interpretations is the entire bypass. The correct invariant is "key presence is the intent signal, value content is the data," and the fix is almost always to replace `truthy(value)` with `key in dict` at the auth-decision boundary.
