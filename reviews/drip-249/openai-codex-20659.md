# openai/codex #20659 — Wire MITM hooks into runtime enforcement

- URL: https://github.com/openai/codex/pull/20659
- Head SHA: `ec08b07d5046e2adcbe5bb7d4f6856c0e28c2cfc`
- Stack: parent #18868 (model + config), this PR (runtime enforcement), follow-up #18240 (config tree placement)
- Files: 5 production + tests across `codex-rs/core/src/config/`, `codex-rs/network-proxy/src/{http_proxy,mitm,mitm_tests}.rs`, README updates

## Context / problem

Parent #18868 introduced the `mitm_hooks` config model — typed match (method/path/headers/query) + actions (strip + inject headers, with `secret_env_var` indirection). It did not enforce anything at the request path. This PR wires the runtime side: a hooked HTTPS host must go through MITM (even in `mode = "full"`), inner HTTPS requests are matched against the host's hooks, matching hooks rewrite headers before forward, and a hooked host with **no** matching hook is denied with `blocked-by-mitm-hook`.

## Design analysis

Three coordinated edits, each at the right boundary:

### 1) Activation gate — `core/src/config/permissions.rs:127-134`
```rust
fn profile_network_requires_proxy(network: &NetworkToml) -> bool {
    ...
    || network.mitm == Some(true)
    || network.mitm_hooks.as_ref().is_some_and(|hooks| !hooks.is_empty())
    || network.mode.is_some()
    ...
```
Adds `mitm` and non-empty `mitm_hooks` to the "permission profile requires the managed proxy to start" predicate. Without this, the proxy would silently never start for a profile whose only network policy is hook-based — and the rest of the runtime would be a no-op. Test at `permissions_tests.rs:251-271` constructs a profile with only `mitm: true` + one hook on `api.github.com` and asserts both `config.network.enabled` and `config.network.mitm` flip true.

### 2) CONNECT-time MITM requirement — `network-proxy/src/http_proxy.rs:261-310`
```rust
let host_has_mitm_hooks = app_state.host_has_mitm_hooks(&host).await?;
let connect_needs_mitm = mode == NetworkMode::Limited || host_has_mitm_hooks;

if connect_needs_mitm && mitm_state.is_none() {
    // emit block decision with REASON_MITM_REQUIRED ...
}
```
The predicate widens from `mode == Limited` to `Limited || host_has_mitm_hooks` — exactly the right disjunction (limited-mode HTTPS needs MITM to enforce method clamps; hooked-host HTTPS in any mode needs MITM to inspect the inner request). The `ConnectMitmEnabled(bool)` extension at `:264,346` threads the decision down into the post-CONNECT path so the upgraded socket only enters MITM when the gate said yes — avoids paying the TLS-termination cost on hosts with no policy stake. The `req.extensions_mut().insert(mitm_state)` at `:316` is now gated `if connect_needs_mitm && let Some(mitm_state)` so unhooked hosts in full mode keep the existing pass-through behavior.

Negative test at `http_proxy.rs:1076-1110` is dispositive: `mitm: true` + one hook on `api.github.com` but no `mitm_state` injected → CONNECT to that host returns `403` with `x-proxy-error: blocked-by-mitm-required`. This is the "hooked host without operational MITM = block, not bypass" invariant.

### 3) Inner-request evaluation — `network-proxy/src/mitm.rs:212-414`
The biggest structural change: `mitm_blocking_response()` (which returned `Result<Option<Response>>` — None = pass, Some = block) is rewritten as `evaluate_mitm_policy()` returning a typed `MitmPolicyDecision` enum:
```rust
enum MitmPolicyDecision {
    Allow { hook_actions: Option<MitmHookActions> },
    Block(Response),
}
```
This is the right type: the previous shape couldn't carry "allow, AND apply these header rewrites" — pass meant pass-with-no-mutation. Three arms:
- `HookEvaluation::Matched { actions }` → `Allow { hook_actions: Some(actions) }` — caller applies strip/inject.
- `HookEvaluation::HookedHostNoMatch` → records `BlockedRequest` with `REASON_MITM_HOOK_DENIED`, returns `Block(blocked_text_response(REASON_MITM_HOOK_DENIED))` — fail-closed at the host level.
- `HookEvaluation::NoHooksForHost` → falls through to existing method-policy / local-private checks.

`apply_mitm_hook_actions()` at `:407-414` is the mutation point: iterate `strip_request_headers` then `inject_request_headers`. Strip-then-inject ordering matters when injecting `authorization` after stripping it (the GitHub-token-shim case in the test fixture at `mitm_tests.rs:18-37`) — and that's exactly the order coded.

The old `mitm_blocking_response` is preserved at `:265-272` with `#[cfg_attr(not(test), allow(dead_code))]` as a thin wrapper over `evaluate_mitm_policy`, so existing test call sites compile unchanged. Reasonable migration discipline.

## Risks

- **Header-name comparison is bytewise**, not case-insensitive (`headers.remove(header_name)` where `header_name` is a `String` from config). HTTP headers are case-insensitive on the wire — `rama_http::HeaderMap` typically case-normalizes, but the `MitmHookActions::strip_request_headers` config fixture uses lowercase `"authorization"` and the PR doesn't show a normalization step at config load. Worth verifying `headers.remove(&str)` does case-insensitive lookup against `HeaderMap`'s storage; if not, an upper-case `Authorization` from the client would slip past the strip step and the inject would `headers.insert("authorization", ...)` next to it — leaving two auth headers on the request. Quick fix: parse via `HeaderName::from_bytes(name.as_bytes())` at config load and store the typed `HeaderName`.
- The `connect_needs_mitm` extension is consumed at `http_connect_proxy()` via `upgraded.extensions().get::<ConnectMitmEnabled>()` — if the upgraded path ever drops extensions across a future `hyper`/`rama` upgrade, the predicate silently falls back to `false` and hooks stop applying. A `.expect("ConnectMitmEnabled missing")` (or a more graceful default-deny) would surface this instead of silently bypassing.
- `host_has_mitm_hooks()` is awaited at every CONNECT — if `app_state` holds the hook list behind a tokio `RwLock`, this is fine; if it's a hashmap protected by a `Mutex`, contention scales with CONNECT QPS. Not visible in this diff. Worth a comment.
- README at `network-proxy/README.md:33-47` documents the matcher syntax (`pattern:`, `literal:`, `**` for crossing segments) but the `match.body` field is called out as "reserved for a future release" — good discipline that the field exists in the model but doesn't silently match yet.

## Suggestions

1. Confirm `HeaderMap::remove` on a `String` does case-insensitive lookup; if not, type-store the header names at config load.
2. Add `expect()` on the `ConnectMitmEnabled` extension lookup, or document the soft-default.
3. Add a test for the strip-then-inject ordering invariant against a request that arrives with `Authorization` already set (mixed case, e.g. `Authorization` not `authorization`) so any future case-handling regression is caught.

## Verdict

`merge-after-nits` — correct runtime wiring of the MITM hook model, with the activation gate at the profile-projection layer (so the proxy actually starts), the CONNECT-time gate widened to `Limited || hooked_host`, the inner-request evaluator restructured into a typed `MitmPolicyDecision` enum that can carry hook actions through the allow path, and a fail-closed `blocked-by-mitm-hook` reason for hooked-host-no-match. Wants header-name case-handling confirmation, an `expect()` on the extension lookup, and a mixed-case-Authorization regression test.
