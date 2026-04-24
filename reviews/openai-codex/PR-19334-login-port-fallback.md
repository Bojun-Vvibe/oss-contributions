# PR #19334 — fallback login callback port when default 1455 is busy

**Repo:** openai/codex • **Author:** xli-oai • **Status:** open
• **Net:** +77 / −4

## Change

`bind_server` in `codex-rs/login/src/server.rs` previously gave up
after `MAX_ATTEMPTS=10` retries (with one stale-server cancel
attempt) on `127.0.0.1:1455`. Now, if the default port is still
busy after the cancel + retry budget is exhausted, fall back to
`127.0.0.1:0` (OS-assigned ephemeral port), reset the attempt
counter, and rebuild the OAuth redirect URI from the actually-bound
port. New e2e test occupies port 1455 with a non-codex listener and
asserts the auth URL ends up on a different port.

## Load-bearing risk

The OAuth redirect URI is part of the **registered** redirect set on
the OAuth client side. Most providers require an exact-match against
a pre-registered URI, sometimes with a wildcard port for `localhost`.
This PR builds `redirect_uri=http://localhost:{actual_port}/auth/
callback` from a runtime-decided port, which only works if the OAuth
client is registered with `http://localhost` (any port allowed) — the
RFC8252 native-app pattern. The test exercises a mock issuer, so it
won't catch a real-world mismatch where the production OAuth provider
is registered for an exact `localhost:1455` URI. If that's the case,
the fallback "succeeds" at binding but then the OAuth exchange fails
with a less-obvious `redirect_uri_mismatch` error.

Two more sharp edges:

1. **Cancel suppression after fallback.** The new
   `if !cancel_attempted && !using_ephemeral_fallback` guard means
   once we've moved to ephemeral, we never try to cancel a stale
   server again — but the fallback resets `attempts = 0` and keeps
   looping. On `127.0.0.1:0` you basically can't get `AddrInUse`,
   so this is fine in practice, but the loop structure is now
   subtly stateful and a future change that makes ephemeral binds
   fail (e.g. `EACCES` from a sandbox) would spin for another 10
   attempts uselessly.

2. **Cursor-coexistence framing.** The PR rationale says Cursor on
   Windows already binds 1455. If both apps choose ephemeral
   fallback, both will work — but the user has to launch the OAuth
   flow from the right app. The opened browser sends the auth code
   to whichever process owns the port at that moment. If Cursor is
   on `1455` and codex is on ephemeral `54321`, but the auth provider
   redirects to `1455` (because that's what was originally
   advertised the first time the user signed in to Cursor), codex
   never receives the code. The redirect URI is rebuilt per-flow,
   so this is actually fine — but only if we trust that the OAuth
   provider doesn't cache.

## Concrete next step

Add a test that exercises the ephemeral path against a mock OAuth
provider that **rejects** non-`1455` redirect URIs, and asserts we
get a clear, actionable error rather than a generic
`redirect_uri_mismatch`. If real-world providers can reject, surface
that to the user with "your OAuth client doesn't allow ephemeral
ports — free port 1455 (used by: $(lsof -i :1455)) and retry."

Also: cap the post-fallback retry budget at 1, since ephemeral binds
should always succeed; failing once means something deeper is wrong.

## Verdict

Solid pragmatic fix for the Cursor-collision scenario. The OAuth
redirect-URI registration assumption needs to be documented in the
PR body — otherwise future provider integrations will silently
break.

## What I learned

"Fall back to ephemeral port" is a great pattern for dev tools, but
it tacitly couples to the OAuth client registration policy. Any
codebase that does this should have a comment near the bind site
saying "the OAuth client *must* be registered with `http://localhost`
without a fixed port" — otherwise the implicit contract is invisible.
