---
pr: 19650
repo: openai/codex
sha: bf9880e38039bcf3f332eda2e99355e0db8af961
verdict: merge-after-nits
date: 2026-04-26
---

# openai/codex #19650 — feat: verify agent identity JWTs

- **Author**: efrazer-oai
- **Head SHA**: bf9880e38039bcf3f332eda2e99355e0db8af961
- **Size**: ~3,063 diff lines spanning `codex-login`, `codex-agent-identity`, `app-server`, `mcp-server`, `cli`, `tui`, `exec`, plus the `cloud-requirements` loader.

## Scope

Today, when `CODEX_AGENT_IDENTITY` is set, codex-login trusts an already-decoded local identity blob. This PR moves the trust boundary to a signed RS256 JWT: `AuthManager` becomes async, fetches the backend JWKS for the configured ChatGPT base URL (normalized to `/api/codex/agent-identities/jwks`) only when `CODEX_AGENT_IDENTITY` is present, verifies issuer/audience/`kid`/`exp`, and only then constructs the existing `AgentIdentity` auth record. Process task ids for AgentIdentity are now allocated eagerly during auth load instead of via a `OnceCell`. All existing async-call-site boundaries in app-server, mcp-server, cli, tui, exec, and the cloud loader awaits the new async constructors.

## Specific findings

- **Async churn is wide but mechanical.** `AuthManager::new`, `::shared`, `::shared_from_config`, and reload paths all become `async`. Every caller already lives inside an async boundary, so the propagation is large but unsurprising. Two things to check during review: (a) any test that builds an `AuthManager` synchronously will break — confirm the test surface compiles cleanly under the new signatures, and (b) any cross-thread `Arc<AuthManager>` that was previously constructed under a sync lock now needs the constructor to be re-entered under a tokio `block_on` only at process bootstrap, never inside a request path.
- **JWKS fetch is gated on `CODEX_AGENT_IDENTITY` being present** — good. Regular API-key, ChatGPT token, and stored-auth flows do not pay the network cost. Verify the gating is at the *outermost* check (env var present), not after a successful key parse, so a malformed env value doesn't still trigger a JWKS round-trip.
- **JWKS URL derivation** — `/api/codex/agent-identities/jwks` is normalized off the configured ChatGPT base URL. Normalize defensively: trim trailing `/`, reject `file://` and non-HTTPS schemes, and reject any base URL whose host is `localhost`/`127.0.0.1` unless an explicit env opt-in is set (avoids a malicious local proxy injecting a JWKS).
- **Verification requires RS256 + issuer + audience + matching `kid` + unexpired claims** — that is the right minimum. Three additions worth considering: (1) reject tokens whose `nbf` is in the future, (2) enforce a maximum lifetime (e.g., 24h) regardless of `exp` so a leaked long-lived token has bounded blast radius, (3) require `typ == "JWT"` to defend against header confusion.
- **JWKS caching** — the PR description does not say whether JWKS responses are cached. If every `AuthManager` reload re-fetches, a backend JWKS outage instantly breaks every codex client; if responses are cached forever, backend rotation is silently late. The right answer is a TTL with a short re-fetch on `kid` miss (lazy refresh on unknown `kid`).
- **Eager AgentIdentity task-id allocation at auth load** — this surfaces startup failures explicitly instead of leaking a panic on first use through `OnceCell::get_or_init`. Right call. Make sure the failure mode is a typed error (`AgentIdentityInitError`) with the underlying reason rather than a generic anyhow string, so app-server can map it to a specific HTTP status.
- **Backwards-compat path** — does the PR still accept the legacy already-decoded local identity blob, or is RS256 JWT now the only way? If both are supported, the legacy path needs an explicit `#[deprecated]` with a removal target. If JWT-only, that's a breaking change that should be called out in the PR title/body.
- **`cloud-requirements` becoming async** — fine, but the loader's previous sync API may have been imported by external crates. If `codex-cloud-requirements` is published, this is a semver-major signature change and needs a CHANGELOG entry.

## Risk

Medium. The cryptographic primitives (RS256, JWKS, claim verification) are well-understood and the PR routes them through a focused `codex-agent-identity` module; the bulk of the diff is mechanical async-propagation. The risks live at the edges: JWKS URL derivation hygiene, JWKS caching policy, and clean error mapping for app-server callers. A failure in any of those becomes a hard auth outage for every client.

## Verdict

**merge-after-nits** — (1) document and lock down JWKS URL derivation (HTTPS only, no localhost without opt-in, trailing-slash normalization); (2) state the JWKS caching policy (TTL + lazy refresh on unknown `kid`); (3) add `nbf`, max-lifetime, and `typ == "JWT"` claim checks; (4) make `AgentIdentityInitError` a typed enum, not stringified; (5) state explicitly whether the legacy local identity blob path remains and, if so, mark it deprecated. The architecture is the right one and the async churn is correctly scoped — these are policy and hardening nits, not redesigns.
