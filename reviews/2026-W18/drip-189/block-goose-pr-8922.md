# block/goose#8922 — add encrypted Nostr session sharing

- PR: https://github.com/block/goose/pull/8922
- HEAD: `6c4c3eaa8c9f905d68d765e83a3ac2224a6e14d3`
- Author: callebtc
- Files changed: 13 (+1459 / -16) — new `crates/goose/src/session/nostr_share.rs` (+370),
  `goose-cli/src/{cli,commands/session}.rs` (+145), `goose-server/src/routes/session.rs` (+104),
  Cargo.lock (+363/-1), UI desktop wiring + sessionLinks (+295), openapi.json (+148).

## Summary

Adds a Nostr-based encrypted session-sharing layer. The flow:

1. `goose share` exports the session JSON, encrypts it with NIP-44 v2,
   publishes the ciphertext as kind-30278 Nostr events to a configurable
   list of relays (default: `relay.damus.io`, `relay.primal.net`,
   `nos.lol`, `relay.nostr.band`), and emits a deeplink containing the
   `nevent` (event reference) plus the decryption key.
2. The receiving Goose client (CLI or desktop UI) parses the deeplink,
   fetches from any of the relays, decrypts, and restores the session.

Pure additive feature — no existing session paths change. The diff
introduces a `NostrPublisher`/`NostrFetcher` trait pair so the network
layer can be mocked in tests, plus a `LiveNostrClient` for production.

## Cited hunks

- `crates/goose/src/session/nostr_share.rs:11-12` — `EVENT_KIND: u16 = 30278`
  and `CONFIG_RELAYS_KEY: &str = "GOOSE_NOSTR_RELAYS"`. Kind 30278 is in
  the parameterized-replaceable range (30000-39999) — fine for a "share
  this session" semantic where re-sharing replaces, but the choice
  isn't documented. Worth a NIP-1 reference.
- `crates/goose/src/session/nostr_share.rs:14-19` — `DEFAULT_RELAYS` is a
  hardcoded list of four public relays. Operators in air-gapped or
  enterprise environments will need to override via
  `GOOSE_NOSTR_RELAYS`; the env var is documented but the failure mode
  (no relays reachable → silent degraded experience) deserves an
  operator-facing warning.
- `crates/goose/src/session/nostr_share.rs:21-37` — `NostrShare` (the
  publisher's return value) and `ParsedShareLink` (the fetcher's parsed
  deeplink) are both `Clone, PartialEq, Eq` — good for testability. The
  trait pair `NostrPublisher` and `NostrFetcher` at `:39-47` correctly
  separate the two halves so mocks can verify publisher behaviour
  without touching the fetch path.
- `crates/goose/src/session/nostr_share.rs:55-72` —
  `LiveNostrClient::publish` connects with `Duration::from_secs(8)`
  timeout, calls `client.send_event_to(...)` against all relays in
  parallel, and `if output.success.is_empty()` returns
  `Err(anyhow!("Failed to publish session to any Nostr relay: {:?}", output.failed))`.
  Correct partial-success semantics — one relay accepting is enough — but
  the error message dumps `output.failed` verbatim which may include
  relay-side error text containing the event ID. Probably fine but
  worth a redact pass before logging.
- The `nip44` import at `:6` and `chacha20poly1305 0.10.1` + `bip39 2.2.2`
  + `bech32 0.11.1` Cargo.lock additions — the crypto stack matches NIP-44
  v2 (ChaCha20-Poly1305 AEAD with HKDF-derived per-message keys) which is
  the right choice for ephemeral session-share. Need to confirm the
  upstream `nostr` crate's `nip44` impl has been audited / has a known
  good test-vector match — this is the kind of cryptographic primitive
  where a subtle implementation bug (e.g. nonce-reuse on retry) would
  silently leak session contents.
- ~1459 LOC PR for a brand-new feature on a privacy-sensitive surface
  (session contents include user prompts, model outputs, and tool-call
  results). The diff touches 13 files spanning Rust core, server routes,
  desktop UI, and OpenAPI generation — broader than ideal for a single
  feature PR.

## Risks

- **Crypto correctness.** NIP-44 v2 is well-specified but the `nostr_sdk`
  crate's implementation hasn't appeared in this codebase before. Need a
  reviewer with crypto background to confirm (a) per-message ChaCha20
  nonces are derived correctly (NIP-44 uses HKDF-Expand with the
  conversation key + per-message salt), (b) the AEAD MAC is verified
  before any plaintext is exposed, and (c) the conversation key is
  zeroized on drop. The diff doesn't show the NIP-44 wrapper code so
  these can't be verified from the visible slice.
- **Default-relay privacy.** The four hardcoded relays receive the
  ciphertext + the publisher's pubkey. While the ciphertext is opaque,
  the *act* of publishing reveals that an account is sharing *something*
  to four third-party operators. Worth a one-line note in the docs that
  using the default relays leaks publish-time metadata to external
  parties; air-gapped operators should override.
- **Decryption key in the deeplink.** The deeplink contains both the
  `nevent` reference and the decryption key. Standard practice for
  end-to-end-encrypted "share-by-link" but means the link in clipboard
  history / Slack DM logs / browser history is sufficient to decrypt for
  the lifetime of the relay-stored event. Worth a UX warning at
  share-time and a "rotate / revoke" follow-up issue.
- **Session contents safety.** Session JSON includes prompts and tool
  outputs which can include API keys (model providers, secrets that
  flowed through the conversation), filesystem paths (PII-shaped), and
  internal hostnames. The encryption protects in-transit, but a user
  sharing a session to debug a problem can leak credentials they didn't
  realise were in the conversation. A pre-share scrubber (or at least a
  confirmation dialog naming what's about to be uploaded) would be
  defence-in-depth.
- **Wasm-bindgen + tokio dependency footprint.** Cargo.lock additions
  include `gloo-timers`, `wasm-bindgen-futures`, `web-sys` — pulled in
  transitively by `async-wsocket`. These add compile-time + binary-size
  cost on non-WASM targets where they're never used. Worth a feature-flag
  audit on the `nostr-sdk` dep.

## Verdict

**needs-discussion**

## Recommendation

Hold for (1) a crypto-aware reviewer pass on the NIP-44 v2 integration
to verify nonce derivation, MAC-before-plaintext, and key zeroization,
(2) a privacy/UX review of the share flow — at minimum a clear
confirmation dialog naming what's about to be encrypted-and-published
(prompts, tool outputs, filesystem paths) and a doc note that default
relays leak publish-time metadata to external operators, (3) a
follow-up issue tracking pre-share secret-scrubbing, and (4) a
Cargo.toml feature-flag audit so `nostr-sdk`'s WASM transitive deps
don't bloat non-WASM builds. The feature design is reasonable and the
test-trait separation is the right shape, but a 1459-LOC privacy-surface
addition deserves explicit security sign-off before landing on main.
