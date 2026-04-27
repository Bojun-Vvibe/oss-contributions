# openai/codex #19764 — feat: verify agent identity JWTs with JWKS

- **Repo**: openai/codex
- **PR**: #19764
- **Author**: efrazer-oai
- **Head SHA**: ec92ff1eaea0159e2f3575bdacfe86cbb6e385d2
- **Size**: +529 / −115 across 13 files; load-bearing edits in
  `codex-rs/agent-identity/src/lib.rs` (+191 / −46),
  `codex-rs/login/src/auth/auth_tests.rs` (+230 / −14),
  `codex-rs/login/src/auth/manager.rs` (+52 / −10).
- **Stack position**: 3rd in the efrazer-oai auth stack
  (#19762 → #19763 → **#19764** → #19650 future).

## What it changes

Replaces the previous "decode-only, no signature check" agent-identity JWT
load path with a real JWKS-backed verification flow. The previous code at
`agent-identity/src/lib.rs:115-138` accepted an optional
`public_key_base64: Option<&str>` and, if present, validated an
`Algorithm::EdDSA`-signed token with `validate_exp = false` and
`validate_aud = false` — i.e. signature-only, no claim validation. The new
code:

1. **Switches to JWKS lookup keyed by `kid`** at
   `agent-identity/src/lib.rs:145-167`: fetches the backend's JWKS via
   `fetch_agent_identity_jwks`, calls `decode_header(jwt)` to extract `kid`,
   does `jwks.find(&kid)`, builds an `RS256` decoding key via
   `DecodingKey::from_jwk(jwk)`, and verifies with a `Validation::new(RS256)`
   that **requires** issuer `https://chatgpt.com/codex-backend/agent-identity`
   and audience `codex-app-server` (the two new constants at `:34-36`).
2. **Adds `iss`/`aud`/`iat`/`exp` to `AgentIdentityJwtClaims`** at `:67-71`
   so the previous payload-only path also benefits from the `Deserialize`
   contract enforcing claim presence (the empty-`required_spec_claims`
   branch is gone).
3. **Adds `agent_identity_jwks_url`** at `:312-321` with a three-way
   path-style switch: `/backend-api` bases get
   `/wham/agent-identities/jwks`, `/api/codex` bases get
   `/agent-identities/jwks`, and bare `chatgpt.com`/`chat.openai.com` bases
   get the `/api/codex/agent-identities/jwks` suffix appended.
4. **Tightens `normalize_chatgpt_base_url`** at `:345-356` to handle the new
   `/api/codex` base style: no longer auto-strips the `/codex` suffix when
   the URL ends in `/api/codex`, and no longer auto-prepends `/backend-api`
   when `/api/codex` is already in the URL.
5. **Threads JWKS through the auth load path**: `login/src/auth/manager.rs`
   gets +52 lines threading a `chatgpt_base_url: Option<&str>` into the
   AgentIdentity load helpers, and `cli/src/login.rs:207-217` passes
   `Some(&config.chatgpt_base_url)` into the now-async write helper.
6. **Pin tests** at `agent-identity/src/lib.rs:516-740` cover four scenarios:
   payload-only-decode-still-works (legacy path), full verify-with-jwks
   round trip, untrusted-`kid`-rejected, and missing-iss-or-aud-rejected.
   The test `test_jwks` helper bundles a hardcoded RSA test key
   (`test_rsa_encoding_key()` at `:632-664`) with a matching JWK at `:686-694`.

## Strengths

- **Real verification, not just decode.** The previous
  `validate_exp = false; validate_aud = false; required_spec_claims.clear()`
  configuration was effectively "make sure this is a valid Ed25519
  signature on something" — no protection against token replay across
  audiences, no protection against expired tokens, no protection against
  unrelated issuers. The new code at `:158-167` does
  `set_audience(&[AGENT_IDENTITY_JWT_AUDIENCE])`,
  `set_issuer(&[AGENT_IDENTITY_JWT_ISSUER])`, and inserts both into
  `required_spec_claims`. Plus `Validation::new(Algorithm::RS256)` keeps
  the default `validate_exp = true`, so expiry is enforced too.
- **kid-keyed JWKS lookup is the right shape.** Picking the trusted key
  by `header.kid` (rather than the previous "single static EdDSA pubkey")
  means the backend can rotate signing keys without breaking deployed
  clients — the JWKS endpoint just publishes the new + old keys during
  rollover.
- **Issuer/audience constants are the right way to encode the contract.**
  `AGENT_IDENTITY_JWT_AUDIENCE = "codex-app-server"` and
  `AGENT_IDENTITY_JWT_ISSUER = "https://chatgpt.com/codex-backend/agent-identity"`
  are public string contracts with the backend. Hoisting them to module
  constants means a future "rename audience for v2" PR has exactly one
  line to edit.
- **The `agent_identity_jwks_url` switch correctly mirrors the existing
  `agent_identity_biscuit_url` style.** `/backend-api` → `/wham/...`
  vs. `/api/codex` → `/...` matches the codebase's existing path-style
  convention; the test at `:725-736` pins both branches.
- **Stack-cohesive design.** The PR's three preceding dependencies
  (`refactor: make auth loading async`, `refactor: load AgentIdentity
  runtime eagerly`) exist precisely so this verification step has an
  `async` context to do `client.get(jwks_url).await` without blowing
  up the call graph. That's good upstream-PR-ordering hygiene.
- **Negative-path test coverage is thorough.** Untrusted-`kid` rejection
  at `:602-625` and missing-iss-or-aud rejection at `:629-661` both
  pin the *failure* modes that matter, not just the happy path. A
  future "let's relax claim validation for staging" change would fail
  loud against these tests.
- **Test JWKS embeds a hardcoded test RSA key** at `:632-694`. That's the
  right tradeoff for a verification-path test — generating a fresh RSA
  key per test would slow the suite measurably, and the test key is
  clearly labeled "test RSA key should parse" with the matching modulus
  embedded in the JWK.

## Concerns / nits

- **JWKS fetch has no caching.** `fetch_agent_identity_jwks` at `:128-142`
  does a fresh `client.get(jwks_url).send().await` on every auth load. For
  a long-running TUI that re-validates the token on profile switches, that's
  a network round trip per validation. The standard pattern for JWKS is to
  cache by `kid` for ~5–60 minutes with a refresh-on-miss fallback so a
  legitimate key rotation doesn't produce a hard fail-closed window. Worth
  filing as follow-up before this gets exercised by anything other than
  the initial login flow. Today the 10-second `AGENT_IDENTITY_JWKS_TIMEOUT`
  bounds the worst case but doesn't help the cold-cache rate-limit story.
- **The JWKS endpoint is a hard dependency on a specific backend path.**
  If the backend renames `/wham/agent-identities/jwks` → `/auth/jwks` or
  similar, every client breaks until the constant in `:312-321` is updated.
  A version-suffixed endpoint (`/agent-identities/v1/jwks`) or a discovery
  document would decouple. Not a blocker — backends rename rarely — but
  documenting the contract in a doc comment on `agent_identity_jwks_url`
  would help.
- **The `iat: usize` and `exp: usize` field types** at `:67-71` are unusual
  — JWT timestamps are conventionally `u64` (Unix seconds since epoch) and
  `usize` will be 32-bit on 32-bit targets, which would silently overflow
  for tokens with `exp` past Y2038 (3.0 GHz × 32-bit = 4_294_967_295
  seconds ≈ year 2106 in unsigned, but `usize` on a 32-bit Windows build
  would limit to ~year 2106 only with a u32 not i32). The test fixture
  uses `4_000_000_000usize` which works today but is suspicious; switching
  the type to `u64` removes the platform-dependent ceiling.
- **The `decode_agent_identity_jwt(jwt, /*jwks*/ None)` legacy fallback path**
  at `:148-150` still exists for the payload-only call sites. After the
  full stack lands, every caller should be passing `Some(&jwks)`. Worth a
  follow-up to mark this branch `#[deprecated(note = "use the JWKS-verifying
  path; this branch trusts the JWT payload without signature verification")]`
  or to count call sites and inline the conversion.
- **`normalize_chatgpt_base_url` at `:345-352` uses a let-chain
  (`base_url.ends_with("/codex") && !base_url.ends_with("/api/codex") &&
  let Some(stripped) = base_url.strip_suffix("/codex")`)** which requires
  Rust 1.65+ in the 2021 edition feature gate. That's fine for codex-rs
  today but should be cross-checked against the workspace MSRV.
- **The test RSA private key in `test_rsa_encoding_key`** at `:632-661` is
  embedded as a 1024-bit-looking PEM inside a Rust source file. It's
  unambiguously a test-only key (callers are gated to `#[cfg(test)]`), but
  it's worth a comment marking it as such — `// TEST KEY ONLY — never
  used in production; embedded for offline-deterministic JWT signing in
  unit tests` — to head off scanner false-positives and "leaked secret"
  PR comments from future contributors.

## Verdict

**merge-after-nits.** The architectural shift from "decode-and-trust" to
"JWKS-verify-with-issuer-audience-expiry" is the right security posture,
the test coverage hits all four meaningful branches (verify-success,
untrusted-kid, missing-iss-or-aud, payload-only-fallback), and the
stack-ordering with #19762/#19763 is clean. Before merge: (1) flip
`iat`/`exp` from `usize` to `u64` to remove the 32-bit-target ceiling,
(2) add a test-only banner comment to the embedded RSA private key, and
(3) file a JWKS-caching follow-up so the verification round trip doesn't
become a per-validation latency tax once the full stack ships.
