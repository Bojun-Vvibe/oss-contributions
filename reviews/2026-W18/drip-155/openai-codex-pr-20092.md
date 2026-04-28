# PR #20092 — Return None when auth refresh fails

- **Repo:** openai/codex
- **Link:** https://github.com/openai/codex/pull/20092
- **Author:** gpeal (Gabriel Peal)
- **State:** OPEN
- **Head SHA:** `481d50c759ac273aa30acb670d397feffb988f13`
- **Files:** `codex-rs/model-provider/src/provider.rs` (+7/-1), `codex-rs/app-server/tests/suite/v2/account.rs` (+89/-0)

## Context

Today, when codex has stored auth credentials but the refresh attempt fails (token revoked, network error, server-side denial), the provider raises an error and the caller surfaces "log out and log back in" — which doesn't tell the UI it should *redirect to the login page*. The PR's premise is that returning `None` from the refresh path lets the caller distinguish "no auth" (route to login) from "auth refresh exploded" (route to error toast).

## What changed

The 7-line change in `provider.rs` flips one error branch in the refresh path to return `Ok(None)` instead of bubbling. The 89 added lines in `account.rs` are the integration test scaffold exercising the new contract through the app-server route.

## Design concern: lossy semantics

This is the part that needs discussion. Collapsing "refresh failed transiently" (e.g., 503 from the auth server) and "refresh failed permanently" (token revoked, server returned 401) into the same `None` is a real loss of signal. The PR description even concedes this — "ultimately, we should prevent that from happening." But the proposed fix bakes the lossy behaviour into the public contract of the provider. Once callers start treating `None` as "go to login," any transient refresh failure (DNS glitch, captive portal, mid-roll deploy at the auth server) will silently push the user through a forced re-login they didn't need.

A cleaner shape would be three states: `Ok(Some(token))`, `Err(RefreshError::PermanentlyRevoked)` (caller routes to login), `Err(RefreshError::Transient(_))` (caller retries or shows a toast). The cost is one new enum variant; the value is that you don't lie to the UI about whether the token is recoverable.

## Test analysis

The account.rs test should answer: *which* upstream error currently maps to `None`? If it's only the 401-with-`invalid_grant` branch, the lossy concern goes away and this is fine — that's the canonical "token is dead" signal. If it's anything broader (any 4xx, any IO error, any non-2xx), my objection stands. The PR title and description don't make this explicit and the JSON snapshot of the diff doesn't show me the exact branch. The reviewer should ask the author to walk through the chain inside the matched arm.

The two PR commits (`255d497e` "Add a joke to the README" then `481d50c7` "Codex code") are also a tell that the README change is unrelated — it should be split out into its own PR or dropped before merge. Mixing unrelated commits makes git bisect noisier later.

## Risks

If a transient auth-server outage gets classified as "no auth," users with valid refresh tokens get punted to a fresh device-flow login during the outage. Their subsequent successful login might overwrite a still-valid refresh token on disk with a new one. That's not a security hole but it is a friction tax on every transient auth blip.

## Verdict

**Verdict:** needs-discussion

The mechanism is one line and the test exists. The semantics are the discussion: should the provider differentiate transient vs permanent refresh failure, or is `None` always "user must re-auth"? Also: please drop the README joke commit from this PR.

---

*Reviewed by drip-155.*
