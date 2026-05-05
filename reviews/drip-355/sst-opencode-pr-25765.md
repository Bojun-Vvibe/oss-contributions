# sst/opencode PR #25765 — fix: preserve ChatGPT OAuth refresh token

- Repo: `sst/opencode`
- PR: https://github.com/sst/opencode/pull/25765
- Head SHA: `e3788376c051`
- Size: +23 / -11 across 1 file (`packages/opencode/src/plugin/codex.ts`)
- Closes upstream #25757.

## Summary

Fixes a sign-out-on-refresh bug in the ChatGPT OAuth plugin. The previous
refresh path always overwrote the stored refresh token with whatever was in
the token-refresh response — but OAuth refresh responses are not required to
return a new `refresh_token`, and the upstream issuer often omits it. After
restart, the stored auth would then have `refresh: undefined`, the next
refresh attempt would fail, and the session would silently sign out. Now
the plugin only replaces the stored refresh token when the response actually
contains one, and the typing of `TokenResponse.refresh_token` is correctly
loosened to `string | undefined`.

## What I like

- The type fix is the real correctness signal: `TokenResponse.refresh_token`
  goes from `string` to `string?` at `codex.ts:115`, matching what the
  issuer actually returns. With the old type, any code path that read
  `tokens.refresh_token` was lying about non-nullability; the type change
  forces every call site to either fall back or assert.
- The two initial-login call sites (browser flow and headless flow) keep
  the old "we *do* expect a refresh token here" invariant via the new
  `requireRefreshToken` helper at `codex.ts:120-125` — used at
  `codex.ts:524` and `codex.ts:599`. So initial logins still fail loudly
  if the issuer misbehaves; only the *refresh* path becomes lenient.
- The refresh path at `codex.ts:441-457` is the actual fix:
  ```
  const refreshToken = tokens.refresh_token || currentAuth.refresh
  const expires = Date.now() + (tokens.expires_in ?? 3600) * 1000
  ...
  body: { type: "oauth", refresh: refreshToken, access: tokens.access_token, expires, ... }
  ...
  currentAuth.access = tokens.access_token
  currentAuth.refresh = refreshToken
  currentAuth.expires = expires
  ```
  Importantly, the in-memory `currentAuth.refresh` is also updated to
  `refreshToken` (line 456), so subsequent refreshes within the same
  process see the same value the DB sees. Easy thing to miss.
- `expires` is hoisted into a local before being used in two places, which
  prevents the two writes (DB body and in-memory state) from drifting on
  clock skew between the lines.

## Nits / discussion

1. **Import-order churn.** Lines 1-7 are entirely a re-sort of imports
   (alphabetized, `node:timers/promises` moved up, etc.). Functionally a
   no-op but it inflates the diff and makes `git blame` noisier. If the
   repo runs an import-order linter that did this automatically, fine; if
   it's manual, splitting into two commits ("import sort" + "fix") would
   make the security-relevant change easier to bisect later.

2. **`currentAuth.refresh` mutation is correct but undocumented.** The
   `currentAuth` object is presumably shared by-reference with anything
   else holding the auth handle. Mutating its `refresh` field after a
   response with no `refresh_token` is a silent identity step that future
   readers will wonder about. A two-line comment ("preserve previous
   refresh token across refreshes that don't return one") would prevent
   someone from "simplifying" this back into the bug.

3. **Logging.** `log.info("refreshing codex access token")` at line 439 is
   the only log line in the refresh path. Adding `log.debug` when the
   issuer returned no refresh token (and we kept the old one) would be
   useful for diagnosing token rotations from the field. Optional.

4. **Asymmetric strictness between flows.** The initial login paths now
   assert via `requireRefreshToken`, but `exchangeCodeForTokens` itself
   still types its return value as having an optional `refresh_token`.
   That's fine — the assertion lives at the consumer — but it means the
   "we always get a refresh token on initial code exchange" invariant
   isn't enforced at the issuer boundary. If a future refactor inlines or
   replaces `exchangeCodeForTokens`, the assertion can quietly disappear.

## Verdict

**merge-as-is.** Tight, focused fix for a real silent-logout bug; the
type relaxation is the right shape; the in-memory state mirror is
correctly updated; initial-login flows preserve their stricter
invariant. The only meaningful nit is the unrelated import-sort churn,
which doesn't block the fix.
