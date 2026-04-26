# PR #19620 — Escape turn metadata headers as ASCII JSON

- **Repo**: openai/codex
- **PR**: #19620
- **Head SHA**: `5aa7a5c2498a8b6edb70ed4a071e387538772c1a`
- **Author**: etraut-openai
- **Size**: +109 / -4 across 6 files (incl. new `utils/string/src/json.rs`)
- **Verdict**: **merge-as-is**

## Summary

Turn-metadata headers (a JSON-encoded HTTP header capturing
session/workspace/turn context) currently use
`serde_json::to_string(...)`, which produces UTF-8 bytes. HTTP
header values are technically supposed to be ASCII (RFC 7230
explicitly allows ISO-8859-1 only via opaque-octet rules, and
many proxies/SDKs choke on non-ASCII). A user with a
non-ASCII path (here the test uses `repo-東京`) or a non-ASCII
client metadata field (`origin: 東京`) would ship raw UTF-8 in
a header. This PR adds a `to_ascii_json_string` helper that
emits valid JSON with `\uXXXX` escapes for any non-ASCII char.

## Specific changes

- New `codex-rs/utils/string/src/json.rs:1-85` — defines
  `AsciiJsonFormatter` implementing `serde_json::ser::Formatter`.
  The `write_string_fragment` impl walks `char_indices`,
  flushes the ASCII run, then emits `\uXXXX` for each
  non-ASCII character. Standard escape-on-write pattern.
  Builds on serde_json's existing formatter trait — clean
  reuse. The `to_ascii_json_string` public function (presumed
  past line 85, not in head-200) just instantiates the
  serializer with this formatter.
- `codex-rs/utils/string/Cargo.toml:11-13` — adds `serde` and
  `serde_json` to dependencies. Workspace-versioned, so no
  duplication risk.
- `codex-rs/utils/string/src/lib.rs:+2` — `pub use json::*` or
  similar exposing the new helper.
- `codex-rs/core/src/turn_metadata.rs:73` — `to_header_value`
  switches from `serde_json::to_string(self).ok()` to
  `to_ascii_json_string(self).ok()`. The single-character
  diff at the call site is the entire functional change in
  the core crate.
- `turn_metadata.rs:88` — `merge_responsesapi_client_metadata`
  same swap. Both header-producing paths covered.
- `Cargo.lock:2870-2875` — incidental: removes
  `codex-utils-absolute-path` from some other crate's deps
  (unrelated to this PR's main thrust; presumably a cleanup
  that came along).
- `codex-rs/core/src/turn_metadata_tests.rs:15-16` — test
  fixture changes: `repo` directory renamed `repo-東京`,
  added `origin: 東京` client metadata. Then
  `assert!(header.is_ascii())`, `assert!(!header.contains
  ("東京"))` to prove escaping worked, and
  `assert_eq!(json["origin"].as_str(), Some("東京"))` to
  prove the *parsed* value still round-trips correctly. The
  combination is the right contract: header bytes are pure
  ASCII; JSON semantics preserve the original string.

## Risks

Very low. Two minor things:

1. The old `serde_json::to_string` was infallible-on-success
   for `Serialize` types that don't have non-string keys. The
   new path threads through a custom formatter, but
   `to_ascii_json_string(self).ok()` retains the same
   fallible signature — so no change in error-path behavior.
2. **Performance**: every header build now walks every char
   of every string field. Turn metadata is small and
   per-request, so this is fine, but worth a perf note if
   metadata grows.
3. The new helper lives in `codex-utils-string` — make sure
   any other crate building HTTP headers from JSON adopts it
   too (`grep` for `serde_json::to_string` in header-related
   files).

## Verdict

`merge-as-is` — small, targeted fix for a real interop
problem with a clean reusable helper and a test that pins
the contract from both ends (bytes ASCII, semantics preserved).

## What I learned

`serde_json::to_string` doesn't have an `ensure_ascii` flag
the way Python's `json.dumps` does, so the way to get there
in Rust is a custom `Formatter` impl. The RFC 7230 corner —
HTTP header values *should* be ASCII even though the wire
will carry UTF-8 if you push it — is the kind of thing that
works fine until it hits a proxy or an SDK that disagrees.
The escape-at-emit-time pattern is the right defensive move
because the *consumer* of the header doesn't share your
encoding assumptions.
