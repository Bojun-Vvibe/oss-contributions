# openai/codex PR #20663 — Add stdio exec-server listener

- Link: https://github.com/openai/codex/pull/20663
- SHA: `c429fcf77fa0d9e395553a8cc56bb702780961df`
- Author: starr-openai
- Stats: +183 / −26, 4 files

## Summary

PR 1 of a 5-PR stack. Promotes the previously test-only stdio JSON-RPC connection path in `codex-rs/exec-server` to a first-class transport selectable via `--listen stdio` or `--listen stdio://`. Introduces an internal `ExecServerListenTransport` enum (WebSocket | Stdio), reworks `parse_listen_url` to return that enum instead of a bare `SocketAddr`, drops the `#[cfg(test)]` gates from `JsonRpcConnection::from_stdio` and `write_jsonrpc_line_message` so they compile in release builds, and updates the CLI doc string to advertise the new values.

## Specific references

- `codex-rs/cli/src/main.rs` L449: `--listen` doc is updated from `ws://IP:PORT (default)` to `ws://IP:PORT (default), stdio, stdio://`. Matches the new parser surface.
- `codex-rs/exec-server/src/server/transport.rs` L13–L20: new `ExecServerListenTransport` enum is `pub(crate)` and `#[derive(Debug, Clone, Eq, PartialEq)]`. The crate-private visibility is correct for stack PR 1 — exposing it broader could lock the API before the follow-on PRs land. Consider adding `#[non_exhaustive]` so adding a future variant (e.g. unix socket) doesn't break crate-internal `match`es; not strictly required since the type is `pub(crate)`.
- L33: error text updates from `expected ws://IP:PORT` to `expected ws://IP:PORT or stdio`. Good — the user-visible error matches the CLI doc string.
- L46–L60: `parse_listen_url` now early-returns `Stdio` for `"stdio" | "stdio://"`, otherwise parses `ws://`-prefixed input via `socket_addr.parse::<SocketAddr>().map(ExecServerListenTransport::WebSocket)`. The matching is exact-string for `"stdio"` and `"stdio://"`; `"stdio:"` and `"stdio:///"` are rejected. That is consistent with the documented surface but worth a unit test that explicitly asserts the rejection of those near-miss spellings if not already present.
- `codex-rs/exec-server/src/connection.rs` L9–L15: the four `tokio::io` imports (`AsyncBufReadExt`, `AsyncWriteExt`, `BufReader`, `BufWriter`) lose their `#[cfg(test)]` gates. Same for `JsonRpcConnection::from_stdio` (L33) and `write_jsonrpc_line_message` (L296). This is a real ABI/visibility change for the crate: any other crate previously seeing only the non-test surface now sees `from_stdio`. It is `pub(crate)` so external consumers are unaffected, but reviewers should confirm there is no remaining `cfg(test)`-only struct field these now reference at runtime.
- `codex-rs/exec-server/src/server/transport_tests.rs` (4th file in the PR, not shown in the head of the diff) presumably covers the new parse cases; verify it asserts both `stdio`/`stdio://` accept and a near-miss reject.

## Verdict

verdict: merge-after-nits

## Reasoning

Clean transport-foundation PR with a small, well-scoped surface. The `parse_listen_url` enum promotion is the right shape, and dropping `cfg(test)` from `from_stdio` is necessary for stack PRs 2–5. Two non-blocking nits: (1) add (or confirm) parse-rejection tests for `stdio:` / `stdio:///` near misses; (2) consider `#[non_exhaustive]` on `ExecServerListenTransport` so future variants don't churn `match`es across the crate. Listener wiring (the actual stdio accept loop on the `Stdio` variant) is presumably in stack PR 2 — that is the right cut.
