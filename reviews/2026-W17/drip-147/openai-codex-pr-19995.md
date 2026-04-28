# openai/codex #19995 — fix(network-proxy): normalize network proxy host matching

- PR: https://github.com/openai/codex/pull/19995
- Head SHA: `eebaeec607ed3f58e0c51247fd4d05f65faab22f`
- Author: viyatb-oai
- Files: `codex-rs/network-proxy/src/http_proxy.rs` (+4/−1), `codex-rs/network-proxy/src/policy.rs` (+30/−5), `codex-rs/network-proxy/src/runtime.rs` (+17/−1)

## Observations

- Real bug: scoped IPv6 literals can arrive in at least four equivalent textual forms — `fd00::1%eth0`, `[fd00::1%eth0]`, `[fd00::1%25eth0]` (RFC 6874 percent-encoded), and the unscoped `fd00::1`. If policy stores the unscoped form but a request arrives with a scope suffix, the substring/exact-equality match in `policy.rs` sees a different string and the deny rule silently fails to fire. This is a textbook policy-bypass class — equivalent values that don't normalize before keying the policy table.
- Fix shape (correct): strip the scope suffix at the *normalization layer* before it reaches the matcher, so allow/deny/local-address checks all key on the same canonical form. The +30 in `policy.rs` is the canonicalization function plus its test cells; the +17 in `runtime.rs` is the wiring of every entry point through the canonicalizer; the +4 in `http_proxy.rs` is the call-site update at the request-host-extraction boundary.
- Allow/deny/local-address symmetry: PR explicitly notes "keep allow, deny, and local-address checks aligned on the normalized host value." This is the load-bearing invariant — if you canonicalize on deny but not on allow, a scoped form can still bypass the deny by *not matching the allow either* and falling through to a permissive default. All three checks must use the same canonical input.
- Test scope: "focused coverage for scoped IPv6 literal variants in `network-proxy`" — without seeing the cells, the discriminating tests are: (a) `fd00::1` policy entry + `fd00::1%eth0` request → must match (the bug), (b) `[fd00::1%eth0]` request → must match (bracket form), (c) `[fd00::1%25eth0]` request → must match (percent-encoded scope), (d) `fd00::1%eth0` policy entry + `fd00::1%eth1` request → discriminating cell on whether scopes are *significant* or canonicalized away (different interfaces are different network paths even at the same address — strict reading of RFC 4007 says yes-significant; if the PR canonicalizes to bare `fd00::1` then both interfaces collapse to one policy match, which may be what users expect for the "block this address" use case but loses the "block on eth0 only" use case). Want to verify which choice the PR makes.
- Verification: `cargo test -p codex-network-proxy` only — appropriate scope.

## Risks / nits

- The "scope suffix is canonicalized away" choice (if that's what the PR does) is the right default for security policy (block the address, not the interface) but should be *documented* in code comments to prevent a future "tighten scope handling" patch from reintroducing the bypass.
- Percent-encoded scope (`%25eth0`) decoding has historically been a source of double-decode bugs (decode-then-decode-again produces `%eth0` which then re-decodes catastrophically). Worth checking the PR decodes exactly once, and that the decoder handles `%2` (truncated) and `%XX` non-hex correctly without panicking.
- IPv6 zone identifiers can include arbitrary printable ASCII per RFC 4007 (interface names like `eth0`, `en0`, but also `lo0.1`); the canonicalizer should strip everything from `%` to `]` (or end-of-string) without trying to validate the zone-id charset.
- `[fd00::1%eth0]` bracket form: the bracket pair is a URI-syntax artifact, not part of the IPv6 literal. Worth confirming the canonicalizer strips brackets *and* scope, leaving bare `fd00::1`, rather than producing `fd00::1%eth0` (scope-only stripped) or `[fd00::1]` (bracket-only stripped).
- `runtime.rs` +17: I'd want to see whether this is a single canonicalization-helper function plus 1-2 callsite migrations, vs being scattered across multiple entry points. Single source of truth is the right shape; if the canonicalizer is duplicated, future drift is the new risk.
- No fuzz test against `host: <random>` headers — recommend adding a `cargo-fuzz` corpus seed once landed.

## Verdict: `merge-after-nits`

**Rationale:** Correct identification of a real policy-bypass class via canonicalization-asymmetry between policy storage and request matching. Fix shape (canonicalize at the boundary, then key all three checks — allow/deny/local — on the same canonical form) is right. Pre-merge nits: (a) document the "scope is canonicalized away" choice in code, (b) verify single-decode for `%25` percent-encoded scope, (c) verify bracket+scope are both stripped (not just one), (d) verify all three checks really do canonicalize on the same input. Post-merge nice-to-have: fuzz seed against malformed `host:` headers.
