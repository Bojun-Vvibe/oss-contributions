---
pr_number: 26438
repo: BerriAI/litellm
head_sha: 377f3a46f7c23670b1ee3205ccf8d5d460eacea7
verdict: merge-as-is
date: 2026-04-24
---

# BerriAI/litellm#26438 — apply team TPM/RPM to proxy_admin acting on behalf of a team

**What changed.** Two coordinated edits in `litellm/proxy/auth/handle_jwt.py`:

1. Header resolution (`get_team_id_from_header`) is moved **above** `check_admin_access` in `auth_builder` (lines 1502–1545). Previously, admin-scope tokens short-circuited before the `x-litellm-team-id` header was read, so admins acting on behalf of a team got `team_id=None` and bypassed team rate limits.
2. `get_team_id_from_header` gains a `bypass_allowed_check` flag (line 42) — admin-scope JWTs aren't required to list `groups`, so the existing `allowed_team_ids` membership check would 403 a perfectly valid admin request. Bypass is enabled exactly when `is_admin_scope = jwt_handler.is_admin(scopes=scopes)` (line 1503).
3. `check_admin_access` now accepts `team_id` / `team_object` and threads them into the returned `JWTAuthBuilderResult` (lines 941, 944).
4. `litellm/proxy/auth/user_api_key_auth.py` lines 813–823 add `team_tpm_limit` and `team_rpm_limit` to the user-key auth result so the rate limiter actually reads them downstream.

**Why it matters.** This is the bug fix where the comment is the spec: an admin saying "rate-limit me as if I were team X" must produce team X's limits, not unlimited. Reordering header resolution + carrying the team object through the admin path is the only correct fix; the alternative (post-hoc team lookup after admin check) duplicates code.

**Concerns.**
1. **Test coverage is excellent.** Three new tests in `tests/test_litellm/proxy/auth/test_handle_jwt.py` (lines 1500–1668): admin+header (positive), admin+no header (regression guard for old behavior), admin-scope+empty groups+header (regression guard against the 403 the bypass flag fixes). Together they cover both directions of the reordering.
2. **`bypass_allowed_check` is a one-line semantic decision worth a single-line comment at the call site, not just at the def.** The reader of `auth_builder` sees `bypass_allowed_check=is_admin_scope` and has to jump to read why; inline `# admins have universal route access; team_ids list isn't required in their JWT` would be kind.
3. **Behavior change for non-admin path is zero.** The `elif not team_id:` (line 1544) preserves the old "Get team with model access" branch verbatim — the admin path now just runs *before* it. Good shape.
4. **`team_tpm_limit` / `team_rpm_limit` propagation in `user_api_key_auth.py`** is the only edit outside JWT handling. It's defensive (`if team_object is not None else None`) and matches the surrounding pattern.

Targeted bug fix with strong regression coverage. Land.
