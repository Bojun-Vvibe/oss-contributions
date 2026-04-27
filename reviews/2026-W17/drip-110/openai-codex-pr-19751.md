# openai/codex PR #19751 — opaque app-server identity key, plumbed to remote contracts

- **PR**: https://github.com/openai/codex/pull/19751
- **Author**: @bryanashley
- **Head SHA**: `01b7621d3791a2c93f710ffe7fb2772556ed9818`
- **Size**: +259 / −12
- **Files**: 14, including new `codex-rs/app-server/src/identity_key.rs`, plumbing through `codex_message_processor.rs`, `in_process.rs`, `lib.rs`, `main.rs`, `message_processor.rs`, plus integration tests in `tests/suite/v2/{mcp_resource,remote_thread_store}.rs` and forwarding hooks in `thread-store/src/remote/{list_threads,mod}.rs`.

## Summary

Adds an opaque `IdentityKey` newtype that the standalone app-server binary (and `codex app-server`) accepts via `--identity-key`, threads it through startup → message processor → `RemoteThreadStore`, and forwards it as **binary** gRPC metadata when the remote thread store talks to its contract implementation. The header constant is re-exported so downstream remote contract implementations can read it.

## Verdict: `merge-after-nits`

The shape is right: opaque bytes (not a UTF-8 string), Unix-byte-safe `OsString` conversion, no logging of the key value anywhere I can see, and the tests cover the round-trip including the "argv contains 0x00 / 0xFF" case. Three things hold this back from `merge-as-is`:

1. `--identity-key` accepts the value as an OS argument, which means it ends up in `ps`/`/proc/$pid/cmdline` for any local user. For an "opaque tenant identifier" that's mostly fine, but if downstream contracts ever treat the identity key as a *secret* (the field name doesn't say "non-secret"), this is a footgun. A `--identity-key-file <path>` companion option is the standard cure.
2. In `codex_message_processor.rs:775`, the destructured field is bound as `identity_key: _identity_key` — i.e., received but not yet used. That's defensible for a stack PR (subsequent PRs in the stack will consume it), but the `_` prefix should be a `// TODO(stack): consumed by #19752` comment, or readers will assume it's dead code.
3. The header name being re-exported as a string constant for downstream is the right call, but I want to see *what* the constant is named to confirm it has an `x-` prefix and a tenant-ish namespace, not something generic that will collide with another header in a remote contract integrator's ecosystem.

## Specific references

- `codex-rs/app-server/src/identity_key.rs:1-24` — `IdentityKey { bytes: Vec<u8> }` with `from_bytes`, `from_os_string`, `as_bytes`, `into_bytes`. Newtype pattern is correct; no `Display` impl, no `Debug` redaction wrapper. Adding `#[derive(Debug)]` writes the raw bytes to logs if anyone ever `dbg!(identity_key)` — consider a manual `Debug` impl that prints `IdentityKey(<{N} bytes>)`.
- `codex-rs/app-server/src/identity_key.rs:30-42` — the `cfg(unix)` branch uses `OsStrExt::as_bytes()` for byte-perfect preservation, while `cfg(not(unix))` lossily round-trips through `to_string_lossy().into_owned().into_bytes()`. That's a contract divergence: a Windows operator passing an identity key with a Unicode replacement char will get a different on-the-wire value than a Linux operator passing the same string. Document the divergence or refuse non-UTF-8 on Windows with a clear error.
- `codex-rs/app-server/src/identity_key.rs:54-61` — the `identity_key_preserves_unix_argv_bytes` test specifically pokes the `\x00\xff` case and is gated on `cfg(unix)`. Good — that's the exact regression a future "let's just convert through `String`" refactor would re-introduce. The test name is the right load-bearing identifier.
- `codex-rs/app-server/src/codex_message_processor.rs:415,678,772-776` — `IdentityKey` is added to `CodexMessageProcessorArgs` and destructured in the constructor as `identity_key: _identity_key`. As noted above, replace the `_` prefix with a tracking comment.
- `codex-rs/app-server/src/lib.rs` and `codex-rs/app-server/src/main.rs` — the CLI flag is added at the binary boundary. Verify (in CI logs or locally) that `--help` shows the flag with a description warning that the value is *not* a secret and will be visible in process listings.

## Nits

1. Add `--identity-key-file <path>` and document that `--identity-key <value>` is convenient but visible in `ps`.
2. Replace `identity_key: _identity_key` with `identity_key: _identity_key, // consumed in stack PR #19752` (or whichever number applies).
3. Manual `Debug` for `IdentityKey` that elides the raw bytes.
4. Document or harden the Windows non-UTF-8 path.

## What I learned

Threading "tenant identity" through a multi-process / multi-binary Rust workspace cleanly almost always lands on a newtype-of-`Vec<u8>` with `OsString` ingest, exactly because argv on Unix is bytes-not-strings. The interesting design decision here is *not* logging the key and *not* hashing it on the way through — staying opaque end-to-end is usually the right call when the key is going to land in gRPC metadata that some downstream operator's audit log will pick up. Worth copying the `cfg(unix)` test name verbatim if I ever need to do this in another codebase.
