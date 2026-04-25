# BerriAI/litellm #26468 — [WIP] Add endpoint for bulk key updates for team

- **PR:** https://github.com/BerriAI/litellm/pull/26468
- **Head SHA:** `4fdda3ea35bdbaa7a60a3e59ad3e193200dcd98c`
- **Files changed:** 4 — `litellm/proxy/_types.py` (+2), `litellm/proxy/management_endpoints/key_management_endpoints.py` (+211/−19), `litellm/types/proxy/management_endpoints/key_management_endpoints.py` (+72/−1), and tests (+468/−10).

## Summary

Adds `POST /team/key/bulk_update`: apply a single `update_fields` payload to many keys in one team, selected by either `key_ids` or `all_keys_in_team=True`. Refactors `_process_single_key_update` to take a fully-constructed `UpdateKeyRequest` (instead of building it inside the function from a `BulkUpdateKeyRequestItem`), letting both the existing `/key/bulk_update` and the new endpoint share the same per-key path. Includes a fail-fast team-permission check and a per-key try/except that collects partial failures into `failed_updates`.

## Line-level call-outs

- `key_management_endpoints.py:220-232` (the `all_keys_in_team` branch) — uses `take=MAX_BATCH_SIZE + 1` and rejects when the result exceeds 500. That's the right bound-detection idiom and avoids a separate `count()` round-trip. Good.
- `:237-240` — when `all_keys_in_team=False`, `requested_tokens = list(data.key_ids)` is the *requested* set, not the *existing-in-team* set. The intersection check happens inside the loop at `:272-276` via `if token not in existing_by_token: raise 404`. That's correct, but the resulting `failed_updates` entry from `_build_failed_team_key_update(token, exception, existing_key_row=None)` will leak the caller-supplied `token` value back into the response. If the caller passes guessed/probe tokens, the response body confirms which ones don't exist in this team — a mild enumeration oracle. Consider returning a single aggregate "N keys not found in team" rather than per-token 404s in `failed_updates`.
- `:252-262` — fail-fast permission check uses `existing_keys[0]` as the auth subject. Correct given that all `existing_keys` are filtered by `team_id == data.team_id` at `:222`/`:238`, so any one of them carries the same team. One subtle issue: if `data.all_keys_in_team=True` and the team has zero keys, `existing_keys` is `[]`, the `if existing_keys` guard skips the check, then `:242-246` raises `404 "No keys found for team"`. So a non-admin caller probing arbitrary `team_id`s gets a 404 distinguishing "team doesn't exist / has no keys" from "team has keys but you can't touch them". That's another mild oracle — a 403 in both cases would be more conservative.
- `types/.../key_management_endpoints.py:340-378` — `KeyUpdateFields` extends `KeyRequestBase` and then declares a `model_validator(mode="after")` that `reject_per_key_or_scope_fields` for `key`, `key_alias`, `team_id`. Good defensive layering — prevents a caller from rebinding all keys to a different team in one request. Consider adding `user_id` to that forbidden list too: rebinding ownership in bulk to another user is the same shape of risk as rebinding `team_id`.
- `:356-363` — `validate_temp_budget`: requires both `temp_budget_increase` and `temp_budget_expiry` together. Mirrors the per-key validator. Correct.
- `key_management_endpoints.py:264-265` — `existing_by_token = {row.token: row for row in existing_keys}` plus `update_field_dict = data.update_fields.model_dump(exclude_unset=True)`. The `exclude_unset=True` is critical: it means a `KeyUpdateFields` payload that only sets `max_budget` won't accidentally clear `metadata`/`tags` to `None` for every key in the team. Worth a one-line code comment naming that contract — it's load-bearing.
- `key_management_endpoints.py:282-291` — per-key call passes the constructed `UpdateKeyRequest` into `_process_single_key_update`. The refactor at `:38-50` to take an `UpdateKeyRequest` directly (rather than a `BulkUpdateKeyRequestItem`) is clean and removes a duplicated `UpdateKeyRequest(...)` construction inside `_process_single_key_update`. That's the right shape.
- `:121-147` `_build_failed_team_key_update` — uses `model_dump`/`dict` introspection. Pops `token` from `key_info`. Misses popping any other token-shaped field (e.g. `user_id` may be considered identifying). Probably fine for now.
- General: this is marked `[WIP]` in the title and the PR body says "will add screenshots". Not blocking review of the code shape, but marker should drop before merge. Test coverage (+468 lines) is substantial — that's the right ratio for a permission-sensitive bulk endpoint.

## Verdict

**merge-after-nits**

## Rationale

The endpoint shape is right: scope guard at the top (`team_id`), explicit selection (`key_ids` xor `all_keys_in_team`), bounded batch (500), and shared per-key path so the new code can't drift from `/key/bulk_update` semantics. The `KeyUpdateFields` validator that rejects `key`/`key_alias`/`team_id` from the broadcast payload is the kind of guardrail that prevents the obvious foot-guns. Two enumeration-oracle concerns (per-token 404 leak in `failed_updates`, and 404-vs-403 distinction for non-admin callers on empty teams) are worth tightening before merge — both are small. Adding `user_id` to the forbidden-broadcast list is one line. Drop `[WIP]`, address those, and this lands clean.
