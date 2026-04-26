# block/goose #8839 — fix(oidc-proxy): enforce exp independently of MAX_TOKEN_AGE_SECONDS

- **Repo**: block/goose
- **PR**: #8839
- **Author**: parasol-aser
- **Head SHA**: f4b8d74e1b3f7b404a456fe612c6726355b37fca
- **Base**: main
- **Size**: +594 / −20 across README (+7/−2), `src/index.js` (+4/−3), and
  test files (+583/−15). Fixes #8832.

## What it changes

In `verifyOidcToken` (`src/index.js:190-203`), `exp` enforcement and
the `MAX_TOKEN_AGE_SECONDS` `iat`-cap are now **both** gates that a
token must pass:

```js
if (!payload.exp || payload.exp < Date.now() / 1000) {
  return { valid: false, reason: "Token expired" };
}
if (env.MAX_TOKEN_AGE_SECONDS && payload.iat) {
  const age = Math.floor(Date.now() / 1000) - payload.iat;
  if (age > parseInt(env.MAX_TOKEN_AGE_SECONDS, 10)) {
    return { valid: false, reason: "Token too old" };
  }
}
```

Previously the two were `if … else if`, meaning configuring
`MAX_TOKEN_AGE_SECONDS` would *replace* `exp` enforcement — a token
with a fresh `iat` but past `exp` would pass.

## Strengths

- Correct security fix: configuring an operational age cap should
  *tighten* validation, not loosen it. The previous
  `if/else if` shape was a real foot-gun where an operator setting a
  20-minute cap was simultaneously disabling the IdP-issued
  expiration check.
- README rewrite (lines 31, 108-118) clearly documents the new
  semantics: "Applied **in addition to** the IdP's `exp` claim,
  never as a replacement" — worth a callout in release notes too.
- 530-line new test file (`token-gates.test.js`) plus 53 lines added
  to `index.test.js` covering all four corners:
  - exp expired + age cap configured → rejected
  - exp valid + iat past cap → rejected
  - exp expired + age cap unset → rejected
  - exp valid + iat within cap → accepted
- `Math.floor(Date.now() / 1000) - payload.iat` (line 197) replaces
  the float-arithmetic `Date.now() / 1000 - payload.iat`. Floor
  prevents sub-second drift from accidentally passing the boundary.
  Small correctness improvement bundled with the main fix.

## Concerns / asks

- **Behavior change for existing operators**: anyone who was
  *intentionally* using `MAX_TOKEN_AGE_SECONDS` to accept
  recently-expired tokens (the old, buggy semantics) will see their
  workflows break post-upgrade. The README explicitly mentions
  "GitHub OIDC issues `exp = iat + 300` ≈ 5 min" — for jobs longer
  than 5 minutes, the old behavior was being relied on. Recommend
  calling this out as a **breaking change** in the changelog and
  guiding affected operators to refresh OIDC tokens mid-job.
- The `parseInt(env.MAX_TOKEN_AGE_SECONDS, 10)` with no
  validation: a malformed env var (e.g. `MAX_TOKEN_AGE_SECONDS="abc"`)
  yields `NaN`, and `age > NaN` is always `false` — silently
  disabling the cap. Worth a startup check or a NaN-aware reject.
- The test renames the old `accepts recently-expired token within
  MAX_TOKEN_AGE_SECONDS` block to a rejection — that's the
  intentional semantic flip. Make sure changelog wording matches:
  "**no longer accepts** expired tokens within MAX_TOKEN_AGE_SECONDS".

## Verdict

`merge-after-nits` — correct security fix with thorough test
coverage. Asks: explicit BREAKING-CHANGE call-out in the changelog
+ a NaN guard on the env-var parse. The semantic flip is the right
call regardless.
