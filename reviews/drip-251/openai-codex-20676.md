# openai/codex#20676 — Fix custom CA login behind TLS-inspecting proxies

- **PR:** https://github.com/openai/codex/pull/20676
- **Head SHA:** `2de397ceb91462f493711c716f9aeb8145a09e25`
- **Stats:** +311 / -14 across 7 files (`codex-client/Cargo.toml`, `codex-client/src/custom_ca.rs`, `codex-client/src/bin/custom_ca_probe.rs`, `codex-client/tests/ca_env.rs`, three TLS fixture PEMs)
- **Refs:** SE-6311 (Experian users behind TLS-inspecting proxy)

## Context
Enterprise users (Experian called out by name) are behind TLS-inspecting proxies that re-sign every cert with an internal CA bundle exposed via `CODEX_CA_CERTIFICATE` / `SSL_CERT_FILE`. The login flow's token exchange was failing because the custom-CA branch in `codex-client` still used reqwest's *default* TLS backend selection (which on this platform stack ends up at native-tls / OpenSSL), and that path didn't reliably pick up the rustls-registered custom roots.

## Design
Three coordinated edits, only the first being the actual fix; the other two are test infrastructure:

1. **Force rustls when a custom CA is configured** — `codex-client/src/custom_ca.rs:266-285` (`build_reqwest_client_with_env`) gains an early branch:
   ```rust
   if let Some(bundle) = env_source.configured_ca_bundle() {
       ensure_rustls_crypto_provider();
       info!(source_env = bundle.source_env, ca_path = %bundle.path.display(),
             "building HTTP client with rustls backend for custom CA bundle");
       builder = builder.use_rustls_tls();
       ...
   }
   ```
   This is the load-bearing line — `builder.use_rustls_tls()` makes the rustls backend explicit *before* the custom roots are registered, so the registered roots actually go to the backend that will perform the handshake. Without it, reqwest's default-feature resolution can still pick the native backend and the custom-root registration ends up on the unused rustls config.

2. **Cargo feature** — `codex-client/Cargo.toml:15` adds `"rustls-tls-native-roots"` to reqwest features so the rustls client *also* preserves system-trust-store roots when a custom CA is added (the union: native + custom). Important: without this, force-rustls would *strip* native roots and break callers whose corporate CA bundle expects the chain to validate against an OS-trusted root.

3. **Subprocess TLS handshake test** — `codex-client/tests/ca_env.rs:+175/-5` adds a real local TLS 1.3 server (rustls server-side, threaded `TcpListener` accept loop, channel-receiver for the request body) signed by `fixtures/test-tls-ca.pem`, plus `bin/custom_ca_probe.rs` upgrades to optionally drive a real HTTPS POST when `CODEX_CUSTOM_CA_PROBE_URL` is set. The probe binary at `:11-46` reads two new env vars (`PROBE_TLS13_ENV`, `PROBE_URL_ENV`), constructs a tokio current-thread runtime, and `client.post(url).body("grant_type=authorization_code&code=test").send().await` — token-exchange-shaped. Test asserts pre-merge-base failure, post-fix success.

## Tests
The new TLS handshake test is the dispositive one — it proves the bug existed and is closed because the same test fails at `merge base 8b07132e09 without the rustls fix`. The body is even shaped like a real OAuth token exchange (`grant_type=authorization_code&code=test`) so it exercises POST-with-body through the full TLS layer, not just a GET.

## Risks
- **`use_rustls_tls()` is unconditional once a custom CA is detected** — callers who specifically need native-tls (smartcard middleware, OS-level cert pinning) lose that option silently. The branch is additive (no custom CA → unchanged default), so the blast radius is limited to "user has CA env set", but worth a one-line opt-out via `CODEX_TLS_BACKEND=native` env for the rare smartcard case.
- **`rustls-tls-native-roots` feature** pulls in `rustls-native-certs` transitively (already a dep per Cargo.toml — line 17 had `rustls-native-certs = { workspace = true }` so this is consistent, no new tree).
- **`bazel-lock-update` was run** per PR body — confirm the lockfile diff is committed and not stale.
- **TLS 1.3 specifically** — `min_tls_version(TLS_1_3)` is set when `PROBE_TLS13_ENV` is set, but the production code doesn't pin TLS version. Some TLS-inspecting middleboxes only do TLS 1.2; want a sentence in PR body that this hasn't been narrowed accidentally.
- **Test fixtures** — three new PEMs under `tests/fixtures/`. CA cert + server cert + server key. Generated with what tooling? If no comment, future maintenance ("how do I rotate this when the test cert expires in 10 years") is a footgun. One-line comment in `test-tls-ca.pem` header (or a generation script) prevents this.

## Suggestions
1. Add `CODEX_TLS_BACKEND` escape hatch (or document explicitly that custom-CA forces rustls and that's intentional).
2. Comment / shell snippet in fixture PEMs documenting cert generation so they can be regenerated when they expire (10-year self-signed is typical but eventually bites).
3. Confirm `info!` log at `:271` doesn't accidentally log the CA cert *contents* (looks like just `source_env` + `path`; double-check no follow-up `info!(?bundle, ...)` was added during review).
4. Consider negative test: invalid custom CA → builder fails *before* network traffic (per the `custom_ca.rs:14-17` doc-comment contract — "fail early with a precise error before the caller starts network traffic"). Worth pinning.

## Verdict
**merge-after-nits** — surgical fix to the load-bearing line, real handshake test that names the failure mode and proves the merge-base regression, but the silent native→rustls switch and the cert-rotation story for the test fixtures need one-line callouts.

## What I learned
The reqwest-feature-flag interaction with backend selection is a recurring footgun — adding `rustls-tls-native-roots` doesn't *select* the backend, it just makes that backend's root-store-loading work; you still need an explicit `use_rustls_tls()` call to force selection. And the right discipline for "force a backend only when the user's config requires it" is to make the branch as narrow as possible (here, gated on `env_source.configured_ca_bundle()`) and to log at info-level which backend was selected and why, so when an enterprise user opens a ticket the log immediately shows `"building HTTP client with rustls backend for custom CA bundle"` and you don't have to guess.
