# BerriAI/litellm#26861 — SCIM: Block virtual keys when user is deprovisioned

- URL: https://github.com/BerriAI/litellm/pull/26861
- Head SHA: `dc123d9f12d81244062761677e57dfa6c85931de`
- Size: +534 / -8 (mostly UI build artifacts; ~80 lines of real logic)

## Summary

Closes a real cross-trust gap: when an upstream SCIM provider marks a user inactive (or deletes them), the user's existing LiteLLM virtual keys continued to authenticate until they happened to be evicted from cache. This PR (a) adds an `_set_user_keys_blocked(user_id, blocked) -> int` helper in `scim_v2.py:337+` that flips `blocked` on every key owned by the user via the prisma client AND invalidates them in the in-memory/redis caches via `proxy_logging_obj` + `user_api_key_cache`; (b) wires `scim_active` metadata onto the user record (`scim_transformations.py:45-65`) and reads it back into the SCIM `active` field; (c) adds a defense-in-depth check in `user_api_key_auth.py:1266-1281` that refuses any cached key whose owning `user_obj.metadata["scim_active"] is False`. The bulk of the diff is `_experimental/out/` UI build artifacts being moved from `*.html` to `*/index.html` (route shape change) — those are mechanical regenerations and not the substance of the PR.

## Observations

- `user_api_key_auth.py:1266-1281` — the defense-in-depth gate is the load-bearing security check. Predicate is `if user_obj is not None and user_obj.metadata is not None and user_obj.metadata.get("scim_active") is False`, which uses **identity comparison `is False`** (not `not user_obj.metadata.get("scim_active")`). This is correct — `is False` distinguishes "explicitly deactivated" from "never set" / "set to None". A user who has never had a SCIM mutation will have `scim_active` absent and pass the gate, which is the intended legacy behavior. Good discipline.
- `_set_user_keys_blocked` at `scim_v2.py:337+` does the right pair of operations (prisma update + cache invalidate via `proxy_logging_obj` and `user_api_key_cache`), but the diff slice doesn't show whether the cache invalidation is awaited or fire-and-forget. If it's fire-and-forget without `task.add_done_callback(...)` exception logging, a redis outage will silently leave stale cached keys — verify in the unsliced diff that the cache-invalidate path either awaits or surfaces exceptions to a logger.
- `scim_transformations.py:45-65` — the `active` field defaults to `True` when `scim_active` is `None`. Comment at `:46-47` explicitly names this as legacy compatibility ("Default to True for users that have never had the flag set, e.g. created before this"). Correct default; without it every pre-existing user would suddenly appear deactivated to the SCIM provider on first read and the SCIM client would attempt to reconcile by deleting them.
- The route-shape change in `_experimental/out/*.html` → `*/index.html` is a Next.js export-mode change that's unrelated to SCIM. It's batched into the same PR for what's presumably a coincidental Next config update — the next reviewer should confirm with the PR author that the HTML rename is intentional and not an accidental git artifact from a local `next build` with different `trailingSlash` config. If it's intentional, it warrants its own PR or at minimum a CHANGELOG note ("admin UI URL shape changed; bookmarks to `/virtual-keys.html` now redirect to `/virtual-keys/`").

## Verdict

**merge-after-nits**

## Nits

- Confirm `_set_user_keys_blocked`'s cache-invalidate path either awaits or has `task.add_done_callback(_log_exc)` so a redis outage doesn't silently leave stale cached keys (matches the pattern from drip-199 #26859).
- Split the `_experimental/out/*.html` → `*/index.html` route-shape change into its own PR or document it in the CHANGELOG.
- Add a regression test for the defense-in-depth path: a `user_api_key_cache`-hit on a key whose owning user has `scim_active: False` must raise.
