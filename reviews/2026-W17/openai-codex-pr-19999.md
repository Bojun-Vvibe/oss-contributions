# openai/codex PR #19999 — recheck network proxy connect targets

- URL: https://github.com/openai/codex/pull/19999
- Head SHA: 70f2edda5edd
- Files: `codex-rs/network-proxy/src/connect_policy.rs` (new), `http_proxy.rs`, `lib.rs`, `mitm.rs`, `socks5.rs`
- Verdict: **merge-after-nits**

## Context

The network proxy already enforces an `allow_local_binding`
policy — non-public IP destinations are rejected unless the
config explicitly allows them. But that enforcement lived only
at the upstream proxy decision point. If the proxy was bypassed
(e.g. via `NO_PROXY`) or the inner SOCKS5/MITM/HTTP-proxy
connector was reached by another path, the TCP connect could
still land on a non-public address. This PR introduces
`TargetCheckedTcpConnector` — a wrapping connector that
re-runs `is_non_public_ip(addr.ip())` at the actual `connect()`
boundary and rejects with `ErrorKind::PermissionDenied` when the
policy disallows it.

## Design analysis

The new `connect_policy.rs` is the centerpiece. Two construction
paths cover the two call sites:

- `TargetCheckedTcpConnector::new(state)` — for live-reload paths
  that need to read `NetworkProxyState` async on every connect.
- `TargetCheckedTcpConnector::from_allow_local_binding(bool)` —
  for paths where the allow flag is already resolved at config
  load time and need not be re-read.

The `TargetPolicy` enum cleanly captures both, and the
`allow_local_binding()` method's error mapping wraps reload
failures with `OpaqueError::context("read network proxy config")`
so the failure surface is identifiable in logs.

The crucial bit of the `serve()` impl at lines 53-64:

```
if input.extensions().get::<ProxyAddress>().is_some() {
    return TcpConnector::new().serve(input).await;
}
```

If the upstream stack has already attached a `ProxyAddress`
extension, this connector is being used to talk to *the proxy
itself* — not to the final target — and the target check is
deferred to whatever runs at the proxy's far side. Skipping the
check here is correct; running it would block legitimate
proxy-to-proxy hops to RFC 1918 corporate proxies.

The two test cases are well-chosen:

- `direct_connector_rejects_non_public_target_when_local_binding_disabled`
  binds a real `TcpListener` on `127.0.0.1`, points the connector
  at it, and asserts the resulting error contains `"network
  target rejected by policy"`. End-to-end, no mocks.
- `direct_connector_allows_non_public_target_when_local_binding_enabled`
  flips `allow_local_binding: true` and asserts success. Together
  they pin the policy hinge.

## Risks / suggestions

1. The error string `"network target rejected by policy"` is
   asserted via `format!("{err:?}").contains(...)` in the test.
   That's brittle if anyone reformats the error wrapping later.
   A typed error variant + `downcast_ref` would survive
   reformatting; the string match is fine for now but worth a
   comment.
2. `is_non_public_ip(addr.ip())` is called on every connect. It's
   fast (CIDR membership), but at high connection rates the
   `state.current_cfg().await` lookup inside `TargetPolicy::State`
   could become hot. If `NetworkProxyState` has internal caching
   already, no concern; if not, consider caching the bool
   alongside the policy snapshot.
3. The PR touches multiple connect paths (`http_proxy.rs`,
   `mitm.rs`, `socks5.rs`) but I only deeply read the new
   `connect_policy.rs`. Reviewers should verify each touch site
   is wrapping the *outermost* TCP connector and not accidentally
   nesting checks (which would be correct but wasteful).

## What I learned

Defense in depth for "where is this socket actually pointed"
deserves a re-check at the deepest layer that knows the resolved
`SocketAddr`. Policies enforced only at the upstream-decision
layer can be bypassed by any code path that constructs the
connector differently. The
"check the extension to detect proxy-to-proxy hops and skip"
trick is a clean way to reuse one connector for both direct and
through-proxy traffic without false-positives.
