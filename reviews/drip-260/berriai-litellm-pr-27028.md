# BerriAI/litellm PR #27028 — Fix runtime policy attachment initialization

- PR: https://github.com/BerriAI/litellm/pull/27028
- Head SHA: `1f3d9a32f926b79a2ae130ee270a0d989621ef85`
- Author: @shivamrawat1
- Size: +57 / -0
- Status: MERGED

## Summary

`PolicyRegistry.add_policy` and `AttachmentRegistry.add_attachment` were updating their internal stores (`_policies`, `_attachments`) without flipping `_initialized = True`. A proxy that started with no static policy config but received runtime-created policies+attachments via the policy builder would still report `is_initialized() == False`, causing the policy engine to short-circuit in `add_guardrails_from_policy_engine` and skip the just-created global policy. Fix is two one-line writes plus a regression test.

## Specific references

- `litellm/proxy/policy_engine/policy_registry.py:228` — `add_policy` now sets `self._initialized = True` after the dict write.
- `litellm/proxy/policy_engine/attachment_registry.py:222` — `add_attachment` same pattern, after `self._attachments.append(...)`.
- `tests/test_litellm/proxy/test_litellm_pre_call_utils.py:2849-2902` — new `test_api_created_global_policy_applies_to_new_key_without_restart`. Setup explicitly resets both registries' `_initialized = False` to model the cold-start case, then asserts the runtime guardrail ends up in `data["metadata"]["guardrails"]` after one round-trip through `add_guardrails_from_policy_engine`. Correct red-then-green shape.

## Verdict: `merge-after-nits`

Diagnosis is correct, fix is the minimal correct one, regression test covers the precise scenario. Two structural concerns keep this off `merge-as-is`.

## Nits / concerns

1. **`_initialized` is being treated as a derived flag, not a source of truth.** Every mutation (`add_policy`, `add_attachment`, presumably `remove_*`, `clear`, bulk-load) now needs to remember to update it. A `@property` that returns `bool(self._policies)` (or `bool(self._attachments)`) would remove the foot-gun entirely. Worth a follow-up — the current PR is the right surgical fix for the bug at hand.
2. **`remove_policy` / `remove_attachments_for_policy` paths don't flip the flag back to `False` when the registry empties.** Probably intended (once initialized, stay initialized — a deliberately-empty config is still "configured"), but worth a one-line code comment so the next reader doesn't add a "fix" for it.
3. **Test cleanup uses a `try/finally` to reset registry internals.** Good. But it directly pokes `_policies`, `_policies_by_id`, `_attachments`, `_initialized` — i.e. it relies on private attribute names that this PR is itself touching. A `policy_registry.reset_for_tests()` helper would isolate tests from internal layout.
4. **No assertion that `is_initialized()` returns True after `add_policy` / `add_attachment`.** That's the actual contract this PR is establishing. The current test asserts the downstream behavioral consequence, which is great, but a direct unit test on the registries themselves (`registry.add_policy(...); assert registry.is_initialized()`) would localize future failures.
