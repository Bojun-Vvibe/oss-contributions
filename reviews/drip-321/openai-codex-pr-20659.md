# openai/codex PR #20659 — Wire MITM hooks into runtime enforcement

- Link: https://github.com/openai/codex/pull/20659
- SHA: `ec08b07d5046e2adcbe5bb7d4f6856c0e28c2cfc`
- Author: evawong-oai
- Stats: +419 / −25, 10 files

## Summary

Stack PR 2 (parent #18868 added the MITM hook config + model). This wires the network-proxy runtime so HTTPS hosts covered by `mitm_hooks` are authoritatively gated by hook evaluation even when `mode = "full"`: hooked hosts force MITM on the `CONNECT`, hooks are evaluated in order with first-match-wins, no-match yields a `403 blocked-by-mitm-hook`, and the request-header mutation actions (strip / inject from `secret_env_var`) are applied to the inner request. Permissions-profile gate is widened so `network.mitm` or non-empty `network.mitm_hooks` is enough to require the proxy.

## Specific references

- `codex-rs/core/src/config/permissions.rs` L127–L136: `profile_network_requires_proxy` now also returns true when `network.mitm == Some(true)` or `network.mitm_hooks` is `Some(non-empty Vec)`. The `is_some_and(|hooks| !hooks.is_empty())` guard correctly distinguishes "explicitly empty list" from "absent". Confirms the proxy starts whenever a hook policy is declared.
- `codex-rs/core/src/config/permissions_tests.rs` L249–L271: new `profile_network_proxy_config_enables_proxy_for_mitm_hooks` test asserts `config.network.enabled`, `config.network.mitm`, and `config.network.mitm_hooks.len() == 1` for a single-hook policy. Good positive coverage; please also add a negative case where `mitm_hooks = Some(vec![])` to lock the empty-list semantics in.
- `codex-rs/network-proxy/src/http_proxy.rs` L82–L83: new `ConnectMitmEnabled(bool)` extension type is the right way to thread the per-CONNECT decision into `http_connect_proxy` without re-deriving it. Tiny nit — consider `#[derive(Default)]` for symmetry with the surrounding extension types.
- L261–L268: `app_state.host_has_mitm_hooks(&host).await` introduces an async lookup per CONNECT. Confirm this is `O(hooks)` over a `RwLock`-guarded `Vec` and not e.g. an HTTP fetch; a regression here would put a syscall on every HTTPS connection. The `error!` + `INTERNAL_SERVER_ERROR` fall-through is correct fail-closed behaviour.
- L270: `let connect_needs_mitm = mode == NetworkMode::Limited || host_has_mitm_hooks;` is the central policy join. Reads cleanly. The follow-on `if connect_needs_mitm && mitm_state.is_none()` block at L272–L308 keeps the existing `REASON_MITM_REQUIRED` audit path and just changes the `mode` recorded from a hard-coded `NetworkMode::Limited` to the actual `mode` — important for audit fidelity in full-mode hooked-host blocks.
- L296–L298: `req.extensions_mut().insert(ConnectMitmEnabled(connect_needs_mitm));` plus the conditional `if connect_needs_mitm && let Some(mitm_state) = mitm_state` is the right gate so `http_connect_proxy` only enables the MITM splice when the policy demanded it (not whenever MITM happens to be globally available).
- L343–L350: `http_connect_proxy` now reads `ConnectMitmEnabled` from extensions instead of re-checking `mode == Limited`. Symmetry with the accept path is good. Edge case to verify: if `ConnectMitmEnabled` is absent (extension lookup returns `None`), `is_some_and(|enabled| enabled.0)` is false and the tunnel is treated as raw — confirm this matches the behaviour for hosts that don't go through `http_connect_accept` (should be impossible in production, but a `debug_assert!` here would catch refactor regressions).
- L1073–L1115: new `http_connect_accept_blocks_hooked_host_in_full_mode_without_mitm_state` test exercises exactly the new semantics — a hooked host in full mode with no `mitm_state` returns a block. This is the highest-value new test.
- `codex-rs/network-proxy/README.md` L8–L138: docs additions correctly enumerate the new `blocked-by-mitm-hook` and `blocked-by-mitm-required` x-proxy-error codes, document first-match-wins ordering, and call out that `mitm = false` plus declared `mitm_hooks` is the surprising combination (hooks force MITM regardless of the global flag, modulo the proxy-startup gate above). The matcher syntax block (`literal:` / `pattern:` prefixes, `**` segment-crossing wildcards) is a meaningful surface change worth its own follow-on doc PR if it isn't already covered.

## Verdict

verdict: merge-after-nits

## Reasoning

Substantive runtime-enforcement change, but well-scoped and well-tested. The policy join (`connect_needs_mitm = limited || hooked`) is correct, the audit-event `mode` fix preserves traceability for full-mode blocks, and the new CONNECT test directly covers the behaviour change. Three non-blocking nits: (1) add an empty-`Vec` negative test to lock the `is_some_and(!is_empty())` semantics; (2) verify `host_has_mitm_hooks` is in-process O(hooks) and not a remote/IO call; (3) consider a `debug_assert!` on the `ConnectMitmEnabled` extension presence in `http_connect_proxy` to catch future plumbing regressions.
