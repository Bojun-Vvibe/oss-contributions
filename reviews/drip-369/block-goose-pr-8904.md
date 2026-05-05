# Review: block/goose PR #8904

- **Title:** fix(oidc-proxy): validate exp independently of MAX_TOKEN_AGE_SECONDS (#8832)
- **Author:** bzqzheng
- **Head SHA:** `2eb4a5d9966f72ea23c67a24f780110c4c5a01f4`
- **Files touched:** 2
- **Lines:** +6 / -3

## Summary

Tightens token validation in `oidc-proxy/src/index.js:195-200`. Prior
behavior: when `MAX_TOKEN_AGE_SECONDS` was configured, the `exp`
claim check was placed in an `else` branch and therefore *skipped*
entirely if the token was within the configured age window — a token
could be `MAX_TOKEN_AGE_SECONDS`-fresh by `iat` but already past its
own `exp` and still pass.

The fix splits the two checks: `iat`-age is still gated by
`MAX_TOKEN_AGE_SECONDS`, but the `exp` check now runs unconditionally
on every token. The accompanying test in
`oidc-proxy/test/index.test.js:232-249` is renamed and inverted:
`"accepts recently-expired token within MAX_TOKEN_AGE_SECONDS"` →
`"rejects expired token even when within MAX_TOKEN_AGE_SECONDS"`,
asserting `401` and `error: "Token expired"`.

## Notes

- Correct security fix. Standard OIDC/JWT semantics require `exp` to
  be honored regardless of any auxiliary freshness policy; the prior
  `else` branch was a real auth bypass class for any deployment that
  set `MAX_TOKEN_AGE_SECONDS`.
- Test inversion is the right shape — existing test was actually
  *enforcing* the bug, so flipping it locks the corrected behavior.
- Diff is minimal and contained to `verifyOidcToken`. Low blast
  radius.
- Linked to issue #8832; reviewers should confirm that no other
  call site relied on the previous "grace window overrides exp"
  behavior.

## Verdict

**merge-as-is**
