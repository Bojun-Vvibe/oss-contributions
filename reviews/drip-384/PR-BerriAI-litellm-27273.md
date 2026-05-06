# BerriAI/litellm#27273 — Fix team member budget enforcement without user row

- **Head SHA**: `4dfe64adb4739db6a32a7deefa491298cf82adad`
- **Stats**: +73 / -1, 2 files

## Summary

`_check_team_member_budget` in `litellm/proxy/auth/auth_checks.py` was gated on `user_object is not None`, so the team-member-budget check would silently no-op whenever the proxy's `LiteLLM_UserTable` row was absent — most notably right after a key was deleted-and-regenerated, where the new key creates a fresh `LiteLLM_VerificationToken` row but the user row may transiently not be loaded into the auth flow. Net effect: a user who had spent $100/$50 on team budget could delete their key, regenerate, and get $50 of fresh runway. The fix is a one-line removal of the precondition: budget enforcement now reads `LiteLLM_TeamMembership.spend` regardless of whether the user row was loaded.

## Specific citations

- `litellm/proxy/auth/auth_checks.py:3517-3521`: removes the `and user_object is not None` clause from the four-way precondition guard. Remaining preconditions (`team_object is not None and team_object.team_id is not None and valid_token is not None and valid_token.user_id is not None`) are sufficient — the budget lookup itself uses `valid_token.user_id` and `team_object.team_id`, not anything off `user_object`. So removing the guard is a strict correctness improvement, not a relaxation.
- `tests/test_litellm/proxy/auth/test_team_member_budget.py:309-381`: `test_team_member_budget_check_blocks_regenerated_key_after_old_key_exhausts_budget` is a thorough regression test — it constructs a `LiteLLM_TeamMembership` with `spend=0.0000002` and `max_budget=0.0000001` (deliberately tiny floats so any floating-point comparison drift would also fail), passes `user_object=None` (the regression precondition), and asserts `BudgetExceededError` is raised with both `test-user-1` and `test-team-1` in the message. The `mock_get_team_membership.assert_any_await(...)` verifies the membership-lookup path actually fires.

## Verdict

**merge-as-is**

## Rationale

Surgical, well-explained fix to a real revenue/abuse bug — "delete + regenerate key resets budget" is the exact kind of silent enforcement gap that's hard to detect in production until a customer complains. The test is excellent: it (a) reproduces the exact scenario (user_object=None + regenerated token + non-zero membership spend), (b) uses sub-cent budget thresholds so any rounding regression would fail, (c) asserts the right async lookup actually happens via `mock_get_team_membership.assert_any_await(...)` rather than just trusting the exception bubble-up. The one-line code change is the minimal correct fix — the previous guard was over-defensive and silently skipped enforcement. The four-way precondition that remains correctly handles the "personal key (no team)" and "no token user_id" cases that `_check_team_member_budget` shouldn't run for. Ship it.
