---
pr-url: https://github.com/BerriAI/litellm/pull/26843
sha: a9db887bdd1e
verdict: merge-after-nits
---

# chore(auth): harden invite-link onboarding token flow

Substantial security hardening of the invite-link `/onboarding/get_token` → `/onboarding/claim_token` flow. Three load-bearing changes:

1. **`get_token` no longer mints a UI session key on first hit.** Previously it minted a long-lived dashboard key the moment the invite link was opened — anyone who could pull the URL out of an email preview pane (link-prefetch, security scanner, MUA inline-image fetcher) silently consumed the invitation. The new flow at `proxy_server.py:12087-12150` returns a *short-lived 15-minute JWT* (`token_type=litellm_onboarding`, claims `invitation_link`, `user_id`, `exp = now + 15min`, signed with `master_key`/HS256) plus the dashboard URL. The actual UI session key is only minted in `claim_token` after the password is set.

2. **Idempotence check widened from `is_accepted is True` to `is_accepted is True or accepted_at is not None`** at `:12071` and `:12283`. This is the right defense-in-depth: a previous bug or partial transaction could have left `accepted_at` populated without flipping `is_accepted` (or vice versa), and treating either signal as "already used" closes the replay window.

3. **`claim_token` wraps the invite-flip + password-write in a Prisma transaction** at `:12309-12339` (`async with prisma_client.db.tx() as tx:`), with the invite-flip done as `update_many(where={id, is_accepted: False})` returning a count — if the count is 0 the link was raced (TOCTOU between the find_unique check above and the write here) and the response is 401, not silent success. Then the user password write happens *inside* the same tx so a partial commit is impossible. The `_rollback_onboarding_invite_claim` helper at `:12176` is the explicit compensating action if the post-transaction `_generate_onboarding_ui_session_token` throws — it sets `is_accepted=False, accepted_at=None` so the user can retry the link.

The test file `tests/test_litellm/proxy/auth/test_onboarding.py` is rewritten with a `_AsyncTx` async-context-manager mock, a new `_make_onboarding_token` helper that mints the JWT in three negative shapes (wrong `token_type`, wrong `invitation_link`, wrong `user_id`) for the new validation gate, and the `claimed=True` arm of `_make_invite` for the second-signal idempotence check. The test docstring at `:5-9` is correctly updated to describe the new contract — "rejects already-used links and returns only a short-lived onboarding token, not a UI session key" — instead of the old "Invalidate the link (prevents abuse)" text on `:12029-12033` of the prior implementation that was a misleading description of what `is_accepted=True` actually meant.

The change from `team_id="litellm-dashboard"` (string literal) to `team_id=UI_TEAM_ID` (imported constant) at `:12211` is a small but real correctness win: the old hardcoded string would have silently desynced if anyone changed the constant on the team-create side.

Three nits:

1. **`master_key` is reused as the JWT signing key** for the onboarding token. That's defensible (only the proxy needs to verify it) but couples two distinct trust contexts — the master key has many other powers, and rotating it now invalidates every in-flight onboarding session. A dedicated `onboarding_token_signing_key` derived via HKDF from `master_key` would let rotation be done independently without breaking active flows.
2. **`exp = get_utc_datetime() + timedelta(minutes=15)`** at `:12099` mints the JWT `exp` as a Python `datetime`, not a Unix timestamp int. PyJWT accepts both but the conventional shape is `int(...timestamp())` — leaving it as a `datetime` works because PyJWT coerces, but makes the claim non-portable to other JWT verifiers a future SSO integration might wire up.
3. **The 15-minute window is hardcoded.** A real customer with a slow email security gateway could plausibly take 20+ minutes to click through (link rewriting, sandbox scan, click-time deferred delivery). The window should be a config setting (`onboarding_token_ttl_minutes`, default 15) so operators can lengthen it without a code change.

## what I learned
"Open the link" is *not* a valid signal of intent — it's done by mail clients, security scanners, link previewers, antivirus URL emulators, and accessibility tools, all of which happen before any human eye sees the URL. Any flow that takes an irrevocable action on link-open is broken; the only safe pattern is "link-open mints a short-lived intent-token, the actual irrevocable action requires a second authenticated request that consumes the intent-token." The transactional invite-flip + password-write is the second half of that pattern: you need the irrevocable mutation and the user-state mutation to be atomic, otherwise a crash in between leaves the link consumed but the password unset, locking the user out permanently with no retry path.
