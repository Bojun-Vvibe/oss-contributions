---
pr: block/goose#8904
sha: 2eb4a5d9966f72ea23c67a24f780110c4c5a01f4
verdict: merge-as-is
reviewed_at: 2026-04-29T18:31:00Z
---

# fix(oidc-proxy): validate exp independently of MAX_TOKEN_AGE_SECONDS (#8832)

## Context

In `oidc-proxy/src/index.js` `verifyOidcToken`, the previous code
had:

```js
if (env.MAX_TOKEN_AGE_SECONDS) {
  const age = ...;
  if (age > parseInt(env.MAX_TOKEN_AGE_SECONDS, 10)) {
    return { valid: false, reason: "Token too old" };
  }
} else if (!payload.exp || payload.exp < Date.now() / 1000) {
  return { valid: false, reason: "Token expired" };
}
```

The `else if` is the bug: when `MAX_TOKEN_AGE_SECONDS` is set, the
`exp` check is *unreachable*. A signed token with `exp` 5 minutes
in the past but `iat` within the configured window is accepted as
valid. The proxy still validates signature and issuer earlier in
the flow, so this isn't an "unsigned token wins" bug, but it is a
clean spec violation: RFC 7519 says `exp` is a hard upper bound on
acceptable token lifetime.

## What's good

- Fix is the minimal one-liner: replace `else if` with `if` so both
  checks run independently. Author resisted the temptation to
  refactor the surrounding validation pipeline, which would have
  blurred the diff.
- The test update at `oidc-proxy/test/index.test.js:232` is the
  most valuable part of the PR. The previous test was literally
  asserting the buggy behavior:
  `it("accepts recently-expired token within MAX_TOKEN_AGE_SECONDS", ...) → expect(response.status).toBe(200)`.
  The new test inverts both the name and the assertion:
  `it("rejects expired token even when within MAX_TOKEN_AGE_SECONDS", ...) → expect(response.status).toBe(401); expect((await response.json()).error).toBe("Token expired")`.
  That's a textbook example of "the test was documenting the bug" —
  having the new test land in the same commit forecloses regression.
- Author calls out that the test file change was *intentional* in
  the PR body ("Updated the existing test that inadvertently
  documented the bypass behavior"). That kind of explicit reasoning
  in a security-adjacent PR is the right operator hygiene — saves
  the next reviewer from wondering why a test went from passing to
  inverted.
- DCO sign-off mentioned in checklist; matches contributor
  conventions for `block/goose`.

## Concerns / nits

- None substantive. A nit: the `reason: "Token expired"` string is
  reused from the old branch. Consumers parsing reasons will not
  see a behavior change, which is good.

## Verdict

`merge-as-is` — security fix, smallest possible diff, test is
inverted in the same commit so the regression can't quietly come
back. This should land on the next patch release.
