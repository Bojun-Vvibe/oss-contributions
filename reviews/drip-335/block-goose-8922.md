# Review: block/goose #8922

- **Title:** add encrypted Nostr session sharing
- **Head SHA:** `d7ef2cf3a6f17369f3d7e842a549c9607e807b4d`
- **Size:** +1495 / -16
- **Files touched (sample):**
  - `Cargo.lock` (large dep tree growth)
  - `crates/goose-cli/src/cli.rs`
  - `crates/goose-cli/src/commands/session.rs`
  - `crates/goose-server/src/openapi.rs`
  - `crates/goose-server/src/routes/session.rs`
  - `crates/goose/Cargo.toml`
  - `crates/goose/src/session/mod.rs`
  - `crates/goose/src/session/nostr_share.rs` (new, primary implementation)

## Critique

Adds end-to-end encrypted session sharing over the Nostr protocol. New crate dependencies pulled in via `Cargo.lock` include `aead`, `chacha20`, `bech32`, `bip39`, `bitcoin_hashes`, `bitcoin-io`, `async-utility`, `async-wsocket`, `atomic-destructor` — i.e. the `rust-nostr` ecosystem plus its transitive crypto + websocket deps. This is a meaningful supply-chain expansion.

Specific lines:

- `nostr_share.rs:1` — `pub const EVENT_KIND: u16 = 30278;` Hard-coded Nostr event kind. Confirm this isn't conflicting with any registered NIP and consider documenting why this kind was chosen.
- `nostr_share.rs` (search-grep view) — defines `NostrPublisher` and `NostrFetcher` traits with a `LiveNostrClient` impl. Good — abstracts the real client behind a trait so tests can mock without spinning up a websocket. Verify the test file actually exercises the mock path.
- `nostr_share.rs` `install_rustls_crypto_provider()` — has both a real impl and an empty `fn install_rustls_crypto_provider() {}` stub (presumably feature-gated). Make sure the feature-gate logic is correct; calling the stub on a platform that needs the provider will fail at runtime with a confusing error.
- `routes/session.rs` — adds `share_session_nostr` and `import_session_nostr` HTTP handlers with `ShareSessionNostrRequest`, `ShareSessionNostrResponse`, `ImportSessionNostrRequest`. These are public-facing APIs; they should be auth-gated. Verify they sit behind whatever middleware protects the rest of `session.rs`. If goose-server can be exposed on a network interface, an unauthenticated `share_session_nostr` could be abused to publish arbitrary content to Nostr relays from the user's identity.
- `commands/session.rs:handle_session_import(input: String, nostr: bool)` — accepts a `String` from CLI. `bech32`-decoded `nevent` strings can be long; ensure no panic on malformed input (use `Result`, not `unwrap`).
- `cli.rs` — adds `parses_session_export_nostr_relays` and `parses_session_import_nostr_link` tests. Good clap-arg coverage.
- `goose/Cargo.toml` — adding the `nostr` SDK is a substantial dep. Consider gating the entire feature behind a Cargo feature flag (`nostr-share`) so users who don't want the crypto stack can opt out and keep their dependency surface small. Otherwise every goose install now pulls bitcoin_hashes, etc.
- Encryption: PR title says "encrypted" but the diff sample doesn't show the encryption primitive choice. The presence of `chacha20` and `aead` in deps suggests ChaCha20-Poly1305. Should be documented in `nostr_share.rs` module docs: which AEAD, who holds the key, how the key is conveyed in the deeplink (presumably the `decryption_key` field on `ParsedShareLink`). If the key is in the deeplink, anyone with the link can decrypt — fine if that's the threat model, but call it out.
- 1495 LOC for a brand-new feature with novel crypto deps and a public HTTP surface deserves a security-focused review pass before merge.

## Verdict

`needs-discussion`
