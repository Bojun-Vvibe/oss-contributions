# openai/codex PR #21105 — [network-proxy] Cover DNS timeout blocking

- Repo: `openai/codex`
- PR: https://github.com/openai/codex/pull/21105
- Head SHA: `09aa423fd649d38c696d14674863a5a42422000b`
- Size: +84 / -1 in `codex-rs/network-proxy/src/runtime.rs`

## Summary

Refactors the existing private `host_resolves_to_non_public_ip`
helper in
`codex-rs/network-proxy/src/runtime.rs:716-748` to extract a
generic, injectable variant
`host_resolves_to_non_public_ip_with_lookup<F, Fut>` that takes a
closure-based `lookup` function and a configurable
`lookup_timeout`. The original public-by-API call site
(`runtime.rs:716-727`) is preserved by delegating to the new
helper with the existing `DNS_LOOKUP_TIMEOUT` and a closure that
wraps `tokio::net::lookup_host`.

Then adds four `#[tokio::test]`s
(`runtime.rs:1387-1448`) that exercise:

1. **DNS lookup timeout → blocked** — closure returns
   `std::future::pending()`, lookup_timeout = 1ms, expects
   `assert!(blocked)`.
2. **DNS lookup error → blocked** — closure returns
   `Err(io::Error)`, expects `assert!(blocked)`.
3. **Private resolution (`127.0.0.1:80`) → blocked** — expects
   `assert!(blocked)`.
4. **Public resolution (`8.8.8.8:80`) → allowed** — expects
   `assert!(!blocked)`.

## What this is really doing

This is a *test-only* PR. No production behavior changes — the
existing call site behaves identically. The value is locking in
the security-critical "fail-closed on DNS timeout/error"
invariant of `host_resolves_to_non_public_ip`. Without these
tests, a future refactor could silently flip a `match … { Err(_)
=> false, Ok(addrs) => … }` and turn a hard fail-closed into a
fail-open, which would be a real proxy bypass.

## What I like

- The dependency injection via `F: FnOnce(String, u16) -> Fut`
  (lines 736–739) is the minimal surface area needed — no
  test-only `cfg` flags, no globals, no mockall.
- The four tests cover exactly the four branches of the
  underlying state machine: timeout, lookup-error, private-IP,
  public-IP. There is no fifth interesting branch.
- `runtime.rs:744-748`: the comment "Block the request if this
  DNS lookup fails. We resolve the hostname again when we
  connect, so a failed check here does not prove the destination
  is public." stays attached to the new injectable code path, so
  the security rationale is preserved next to where someone might
  edit it.

## Nits

1. **Closure signature ergonomics.** The `lookup` closure takes
   `String` (line 738) instead of `&str` because of the implied
   `'static` lifetime of the returned future. That forces the
   production wrapper at lines 720-724 to do
   `lookup_host((host.as_str(), port))` after taking ownership of
   `host: String`. Acceptable, but a one-line comment explaining
   *why* `String` instead of `&str` would save the next reader 30
   seconds of staring.

2. **Test for the IPv4 + IPv6 mixed-resolution case** would be
   nice — e.g. closure returns `[8.8.8.8:80, ::1:80]`. Today's
   `is_non_public_ip` will block on the v6 loopback if it's
   in the list, but the test suite doesn't pin that down. Easy
   add. Not blocking.

3. **`SocketAddr` import.** `use std::net::SocketAddr;`
   (line 9) is added but only used inside `#[cfg(test)]` plus
   the new generic helper signature. Fine — but if the test
   were ever cfg'd off (it isn't, today), the import would need
   `#[cfg(test)]`.

## Verdict

**merge-as-is** — pure test-coverage win on a security-critical
fail-closed path. The injection design is the right shape and
the four tests are exactly the branches that matter. Nits are
purely stylistic and can wait.
