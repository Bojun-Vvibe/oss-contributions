# BerriAI/litellm#26492 — Tighten caller-permission checks on key route fields

- **URL**: https://github.com/BerriAI/litellm/pull/26492
- **Author**: yuneng-berri
- **Head SHA**: `2220f3076ac89bd2a2e3439acf57dcfbec2434c9`
- **Verdict**: `merge-after-nits`

## Summary

Hardens the admin gate on `allowed_routes` and adds a parallel gate for
`allowed_passthrough_routes` (both top-level and nested under
`metadata`) on `/key/generate`, `/key/update`, and
`/key/regenerate` in
`litellm/proxy/management_endpoints/key_management_endpoints.py`. The
key behavioural change is the *re-check* of `allowed_routes` after
`handle_key_type` runs — previously, a non-admin could bypass the gate
because `key_type` derives an elevated bucket (e.g.
`["management_routes"]`) **after** the initial caller-permission check.

## Reviewable points

- `_check_allowed_routes_caller_permission` (around line 459) now
  carves out the safe presets `frozenset({"llm_api_routes", "info_routes"})`
  for non-admins via `if all(r in _NON_ADMIN_SAFE_ALLOWED_ROUTES_PRESETS for r in allowed_routes): return`.
  Correct: `handle_key_type` already produces these literal strings for
  non-elevated buckets, so this preserves backward compat for legitimate
  non-admin flows. The use of `frozenset` for the membership test is
  the right idiom.
- The post-`handle_key_type` re-check at `_common_key_generation_helper`
  (around line 713) is the actual security-critical addition. Without
  it, a non-admin passing `key_type=management` would have the gate
  evaluate the *original* (likely empty) `allowed_routes`, then
  `handle_key_type` would silently expand it to an elevated bucket. Now
  both windows are gated.
- `_check_passthrough_routes_caller_permission` (around line 493)
  correctly checks **both** `data.allowed_passthrough_routes` and
  `metadata["allowed_passthrough_routes"]`. The two distinct error
  messages are useful for debuggability.
- Minor nit: the new check runs in three call sites (`generate_key_fn`,
  `_validate_update_key_data`, `regenerate_key_fn`); the
  `regenerate_key_fn` site (line ~3944) is gated on `if data is not None`
  — good defensive shape, but the other two call sites assume `data`
  is present. Worth verifying via tests that all three converge on the
  same 403 contract.
- Test coverage is not visible in this diff — for a security-tightening
  patch a regression test exercising "non-admin with `key_type=management`
  must 403 even when `allowed_routes` is empty in the request body"
  would be high-value.

## Rationale

The vulnerability is concrete (privilege escalation on key creation/
update/regenerate) and the fix is minimal and consistent across all
three endpoints. The carve-out for safe presets keeps non-admin flows
working. The "nits" are tests, not code-shape concerns.
