# openai/codex PR #20676 — Fix custom CA login behind TLS-inspecting proxies

- PR: https://github.com/openai/codex/pull/20676
- Head SHA: `0f8c59b976c6d93daf5826097ac3d2ccd597a838`
- Author: @jgershen-oai
- Size: +504 / -15
- Status: MERGED

## Summary

When `CODEX_CA_CERTIFICATE` or `SSL_CERT_FILE` is set (enterprise TLS-inspecting proxy scenario), force the shared `codex-client` reqwest builder onto rustls before registering the custom CA roots, and pull in the `rustls-tls-native-roots` feature so native roots + the enterprise CA both end up in the trust store. First commit is the production fix (~10 LOC); second commit is +480 LOC of integration test infrastructure that builds a hermetic local CONNECT TLS-intercepting proxy with runtime-generated CA material.

## Specific references

- `codex-rs/codex-client/Cargo.toml:15` — adds `rustls-tls-native-roots` reqwest feature. Necessary so the rustls path doesn't drop native roots when the custom CA is layered in.
- `codex-rs/codex-client/src/custom_ca.rs:273-284` — `build_reqwest_client_with_env` now calls `ensure_rustls_crypto_provider()` and `builder = builder.use_rustls_tls()` before parsing certs, *only* when a bundle is configured. Critically, the no-custom-CA path is unchanged — default reqwest TLS selection still applies for the majority of users. That's the right scope.
- `codex-rs/Cargo.toml:325-328` — `rcgen` 0.14.7 added with `aws_lc_rs` + `pem` features. Dev-dependency only (`codex-rs/codex-client/Cargo.toml:35`), so no production binary size impact.
- `codex-rs/codex-client/src/bin/custom_ca_probe.rs:14-99` — probe binary gains async runtime, optional `CODEX_CUSTOM_CA_PROBE_PROXY` / `CODEX_CUSTOM_CA_PROBE_URL` / `CODEX_CUSTOM_CA_PROBE_TLS13` env-driven modes. The probe still defaults to the original "build client and exit ok" behavior when no env is set, preserving prior subprocess test contracts.
- `codex-rs/codex-client/src/bin/custom_ca_probe.rs:51-65` — `build_probe_client` routes through the new public `build_reqwest_client_with_custom_ca` when a proxy is configured (so the proxy survives the rustls switch), otherwise keeps `build_reqwest_client_for_subprocess_tests`. Symmetric with the production split.

## Verdict: `merge-after-nits`

The production fix is small, well-scoped, and only kicks in when a custom CA is configured — exactly the constraint you want for a TLS backend swap. The +480 LOC of test scaffolding is hermetic (no inherited CA env, runtime-generated CA, local CONNECT proxy) which is the right shape for this kind of "real-world only" failure. Logging the bundle source (`source_env=…, ca_path=…`) at info is the right diagnostic for support-ticket triage.

## Nits / concerns

1. **`use_rustls_tls()` is silently sticky.** If a downstream caller of `build_reqwest_client_with_env` had configured `.use_native_tls()` on the builder before passing it in, the custom-CA branch silently overrides that to rustls. The change is correct (you can't register rustls roots into native-tls), but the behavior should be documented at the function's contract — currently the doc-comment at `:266-271` only mentions "preserves the caller's chosen reqwest builder configuration", which is now untrue for the TLS backend specifically.
2. **`ensure_rustls_crypto_provider()` is a global side effect.** Calling it from a library function is the right pragmatic choice (the provider is process-wide), but if any other crate in the workspace also installs a default provider, ordering becomes load-bearing. A test asserting that calling `build_reqwest_client_with_env` after some other crate's provider install still succeeds would close that out.
3. **Cargo feature drift.** Enabling `rustls-tls-native-roots` on `reqwest` flips the default crypto provider story for the whole crate. `cargo tree -e features` for `codex-client` should be in CI to catch the case where some transitive dep starts requiring native-tls and pulls in both backends.
4. **The `custom_ca_probe.rs:55` error path lossily formats `error: {error:?}` for `reqwest::Proxy::https`.** Most reqwest errors render fine via `Display`; debug formatting buries the chain inside parens. Minor.
