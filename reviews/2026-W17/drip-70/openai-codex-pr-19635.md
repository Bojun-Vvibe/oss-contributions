# openai/codex#19635 — Fix agent identity runtime auth flow

- **Repo**: openai/codex
- **PR**: [#19635](https://github.com/openai/codex/pull/19635)
- **Head SHA**: `3f008f47b35a`
- **Author**: shijie-oai
- **Base**: `main`
- **Size**: +75 / −22, 3 files

## Context

The agent-identity (AI) runtime auth path had three bugs surfaced by
recent staging:

1. `decode_agent_identity_jwt(jwt, public_key=None)` was using
   `Validation::insecure_disable_signature_validation()` + a dummy
   secret to "decode without verifying" — which works but burns a JWT
   library cycle and risks future hardening of `jsonwebtoken` rejecting
   the path. Cleaner to just base64url-decode the payload directly.
2. `register_agent_task` swallowed non-2xx responses behind
   `.error_for_status()`, throwing away the response body and making
   identity-edge debugging painful.
3. `CloudRequirementsService` was calling `auth.account_plan_type()` on
   every auth, including AgentIdentity — but agent-identity sessions
   don't carry a human bearer token at all, so the plan-lookup call was
   guaranteed to 401 against identity-edge.
4. The agent-identity authapi URL was hardcoded as the chatgpt
   backend-api URL, which is the wrong host — it should be the
   `authapi-login-provider.gateway.unified-*.internal.api.openai.org`
   host with `chatgpt-staging.com` host detection picking the staging
   variant.

## Change

Three coordinated edits:

1. **`codex-rs/agent-identity/src/lib.rs:118-150`** — restructures
   `decode_agent_identity_jwt`. The early-return `let Some(public_key
   _base64) = public_key_base64 else { return
   decode_agent_identity_jwt_payload(jwt); };` cleanly splits the
   verify path from the inspect path, and the new
   `decode_agent_identity_jwt_payload<T: DeserializeOwned>` helper does
   a strict 3-part-only split:

   ```rust
   let (_h, payload_b64, _s) = match (parts.next(), parts.next(), parts.next()) {
       (Some(h), Some(p), Some(s)) if !h.is_empty() && !p.is_empty() && !s.is_empty() => (h, p, s),
       _ => anyhow::bail!("invalid agent identity JWT format"),
   };
   anyhow::ensure!(parts.next().is_none(), "invalid agent identity JWT format");
   ```

   The trailing `parts.next().is_none()` check is the part most
   bespoke JWT parsers forget — it's the correct shield against
   `header.payload.sig.injected` shapes.

2. **`codex-rs/agent-identity/src/lib.rs:171-200`** — replaces
   `.error_for_status().context("failed to register agent task")?` with
   an explicit non-2xx branch that captures the body, truncates it to
   512 chars (with a Unicode-safe `chars().take(512).collect()` rather
   than `bytes().take(512)` — correct), and includes status + truncated
   body in the bail message.

3. **`codex-rs/cloud-requirements/src/lib.rs:332-336`** — adds the early
   return:

   ```rust
   if matches!(auth, CodexAuth::AgentIdentity(_)) {
       // AgentIdentity does not carry a human bearer token, and identity-edge
       // only allowlists task-scoped AgentAssertion calls for the Codex runtime.
       return Ok(None);
   }
   ```

   The comment captures *why* (identity-edge allowlist), not just *what*
   — exactly the kind of inline justification a future reader needs.

4. **`codex-rs/login/src/auth/agent_identity.rs:14-115`** — adds the
   `CODEX_AGENT_IDENTITY_AUTHAPI_BASE_URL` env override, two
   prod/staging constants pointing at the
   `authapi-login-provider.gateway.unified-{7,0s}.internal.api.openai.org`
   hosts, and a `agent_identity_authapi_base_url()` resolver with the
   precedence: env override → `chatgpt-staging.com|openai-staging.com`
   substring match → `localhost|127.0.0.1|[::1]` passthrough →
   prod default.

## Strengths

- **Inline justification on the early-return**: the `cloud-requirements`
  comment is the kind of "this surface intentionally short-circuits"
  documentation that prevents a future contributor from "fixing" the
  early-return as dead code.
- **Strict JWT 3-part check**: `anyhow::ensure!(parts.next().is_none())`
  is the right defense against malformed input. Most one-off JWT
  parsers don't do this and end up accepting `a.b.c.d`.
- **Body truncation uses `chars().take(512).collect()` not bytes**: the
  PR author thought about Unicode boundaries. A bytes-truncated error
  body could panic an upstream `String::from_utf8` consumer; this won't.
- **Test update at line 591** (`alg":"none"` instead of `"alg":"EdDSA"`)
  exercises the new no-verify decode path explicitly — matches the
  surface the new helper unlocks.
- **Env override + substring host detection** is the right pattern for
  the staging/prod split: explicit override always wins, host detection
  is fallback. The localhost passthrough enables local-dev mocking.

## Risks / nits

1. **Substring host detection is fragile**:
   `chatgpt_base_url.contains("chatgpt-staging.com")` will match
   anything-staging.com.example.attacker.com if a user passes a
   sufficiently weird `--chatgpt-base-url`. In practice the upstream
   already validates this URL elsewhere, but a stricter
   `Url::parse(...).and_then(|u| u.host_str())` check that compares
   the host *suffix* would be safer. Worth a follow-up.
2. **PR title says "Fix" but this is really three independent fixes
   plus one feature** (the env override). Splitting into
   `fix(agent-identity): decode unsigned JWT payload directly`,
   `fix(agent-identity): surface registration error body`,
   `fix(cloud-requirements): skip plan lookup for AgentIdentity`,
   `feat(agent-identity): add authapi base URL override` would make
   the bisect history much easier. Big PRs in security-sensitive paths
   are bisect-hostile when one of the four pieces regresses.
3. **No test for the env-override resolver**: `agent_identity_authapi_
   base_url(None)` with `CODEX_AGENT_IDENTITY_AUTHAPI_BASE_URL=https://x`
   set would be a 5-line test. Same for the staging substring match
   and the localhost passthrough.
4. **`unwrap_or_default()` on the body read**: if the body read fails
   (network reset mid-response), the error message will say
   `status: 500: ` with empty body, which is correct but worth a
   `(could not read body)` literal so the operator can distinguish
   "body was empty" from "body read failed".
5. **The `error_for_status` removal also drops the auto-retry semantics
   some HTTP layers depend on**. Codex's `register_agent_task` is a
   one-shot with explicit timeout, so this is fine, but a comment on
   the line clarifying "we capture the body before returning so we
   don't need error_for_status" would future-proof against a refactor
   that adds it back.

## Verdict

**merge-after-nits** — the three fixes are correct and well-justified
inline. Worth splitting into three commits before merge, plus adding
a test for the env-override resolver. The substring host detection is
the only thing that mildly worries me long-term.

## What I learned

The "decode JWT without verifying" pattern shows up everywhere people
need to inspect a token they haven't bootstrapped the public key for
yet (e.g. discovery flows). The right shape is *not* to pass an empty
key + `insecure_disable_signature_validation` to a real JWT lib —
that conflates "I don't have a key" with "I don't care about a key".
The right shape is a separate `decode_payload_without_verifying()`
helper with an explicit name, so every caller has to consciously opt
into the unverified path. This PR's structural split exemplifies that.
