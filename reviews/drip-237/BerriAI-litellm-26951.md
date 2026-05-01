# BerriAI/litellm#26951 — chore(proxy): tighten resource ownership checks

- **PR**: https://github.com/BerriAI/litellm/pull/26951
- **Head SHA**: `e9fb89b90c35b28b2ebbb87d18f739b194f12b8b`
- **Size**: +2483 / -119, 16 files (new `resource_ownership.py` helper, container-endpoints ownership module, skills-handler/store split, two large new test files at 1035 + 435 lines)
- **Verdict**: **merge-after-nits**

## Context

Two ownership-check gaps closed in one PR:

1. **Skills (`litellm_proxy/skills/`)** — `LiteLLMSkillsHandler.create_skill / list_skills / get_skill / delete_skill / fetch_skill_from_db` previously took only `user_id` (or nothing) and dispatched directly to Prisma without checking whether the calling user could actually read or mutate the requested skill. Net effect: any authenticated user could fetch or delete any skill_id by guessing the UUID.
2. **Container endpoints (`proxy/container_endpoints/`)** — new 349-line `ownership.py` module plus 55-line `ownership_store.py` enforcing the same per-resource-owner-scope discipline at create/list/get/delete time.

## What's right

- **Centralized ownership predicate.** New `litellm/proxy/common_utils/resource_ownership.py` (+83 lines) exposes `is_proxy_admin(user_api_key_dict)`, `get_primary_resource_owner_scope(user_api_key_dict)`, `get_resource_owner_scopes(user_api_key_dict)`, `user_can_access_resource_owner(owner, user_api_key_dict)`. Both the skills handler and the container-endpoints handler import from here, so ownership semantics can't drift across the two surfaces.
- **The `_user_can_access_skill_owner` helper at `skills/handler.py:30-43` correctly handles the unowned-resource case.** Three branches: (a) `owner is None` AND user is admin → allow; (b) `owner is None` AND `LITELLM_ALLOW_UNOWNED_SKILL_ACCESS=1` → allow with a `verbose_logger.warning` so the operator sees the legacy-data carve-out; (c) otherwise delegate to `user_can_access_resource_owner`. The env-var carve-out is the right shape for a migration: existing pre-ownership-tracking skills don't get bricked, but the access is logged so an operator can see how often it fires and decide when to flip the default.
- **`list_skills` filter pushdown.** At `skills/handler.py:175-195`, the new code constructs Prisma `where: {created_by: {in: owner_scopes}}` (or the OR-with-None variant when the env-flag is on) so the access check happens at the database, not in Python after fetch. Correct: prevents the "fetch-100, filter-down-to-3" leak of unauthorized rows over the wire even if they're ultimately filtered.
- **Owner attribution on create.** At `skills/handler.py:111`: `owner = get_primary_resource_owner_scope(user_api_key_dict) or user_id` then writes `created_by = owner, updated_by = owner`. The fallback to `user_id` preserves the prior behavior for callers that don't pass `user_api_key_dict` (legacy callers, internal calls).
- **Store-handler split.** New `LiteLLMSkillsStore` at `skills/store.py` (22 lines) wraps Prisma's `litellm_skillstable` table. The handler now calls `store.create_skill(skill_data)` / `store.list_skills(find_many_kwargs)` / `store.find_skill(skill_id)` / `store.delete_skill(skill_id)`. Clean separation: handler owns the access-control predicate, store owns the persistence shape. Test files can mock the store cleanly without mocking Prisma directly.
- **Test mass.** 1035 lines for container-endpoints ownership + 435 lines for skills ownership. Reading the structure: per-method coverage of the admin-bypass branch, the owner-match branch, the owner-mismatch branch, the unowned-resource branch, and the env-var-carve-out branch. This is the correct test density for a security-tightening change.

## Risks / nits

- **The "skill not found" error message at `:218,225,250,257` is reused for the access-denied case.** This is the right call (don't leak existence to unauthorized callers) but the test file should pin the property explicitly: assert that an authorized user looking up a non-existent skill and an unauthorized user looking up an existing-but-not-theirs skill both receive the same `Skill not found: <id>` message with the same exception type.
- **`fetch_skill_from_db` at `:280-291` swallows `ValueError` and returns `None`** including the access-denied case. Skills-injection-hook callers will silently fall through to "skill not found" behavior. Worth a one-line `verbose_logger.debug("Skill access denied for user X")` so a misconfigured deploy doesn't look like a missing skill in logs.
- **`ALLOW_UNOWNED_SKILL_ACCESS_ENV` accepts `{"1", "true", "yes"}` (lowercase).** A user setting `LITELLM_ALLOW_UNOWNED_SKILL_ACCESS=True` (capital T, common Python-stringy habit) would silently NOT enable the carve-out. Either add `.lower()`-stripped match against `{"1","true","yes","on"}` (already does .lower()) or document the accepted values in the env-var name docstring. The current `.lower()` does cover `True` / `TRUE` so this is fine on closer reading; still worth a comment.
- **`get_primary_resource_owner_scope` semantics on multi-team users.** Not visible in this diff but called at create time to attribute ownership. If a user is in multiple teams/orgs, the "primary" choice is silently made — worth a doc-comment in `resource_ownership.py` pinning the precedence rule (team > org > user_id, or whatever it is) so future contributors don't change it accidentally.
- **The `_lazy_openapi_snapshot.json` regenerated diff (+45/-45)** is unreviewable here — assumed to be generated; if there's a make-target that produces it, link to that target in the PR body so reviewers can verify it isn't manually edited.

## Verdict

**merge-after-nits.** Solid security-tightening with the right helper-extraction discipline (one `resource_ownership.py` source-of-truth across skills + container endpoints), filter-pushdown at the DB layer, and proportional test mass. Address the access-denied-vs-not-found logging parity and the multi-team primary-scope doc-comment before merge; the rest are cosmetic.
