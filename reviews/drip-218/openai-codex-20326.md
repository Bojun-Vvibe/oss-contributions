---
pr-url: https://github.com/openai/codex/pull/20326
sha: 245b70173128
verdict: merge-as-is
---

# Remove core protocol dependency [3/5]

Predecessor in the same Sapling stack as #20327, doing the actual work of moving the canonical permission-profile / filesystem-permission / network-policy types out of `codex_protocol::*` and into `codex_app_server_protocol::*`. Net diff `+4865/-8074` — a real refactor, not a delete. Two representative sites:

- `codex-rs/cli/src/main.rs:533-540`: replaces `codex_protocol::protocol::FinalOutput::from(token_usage)` formatting with a direct `token_usage.to_string()`, retiring a wrapper type whose only job was to bridge the old protocol module's `Display` impl. Test import at `:1688` switches from `codex_protocol::protocol::TokenUsage` to `codex_tui::TokenUsage` — the type is now re-exported from the consumer crate.
- `codex-rs/tui/src/additional_dirs.rs:46-130`: tests that previously constructed `PermissionProfile::External { network: NetworkSandboxPolicy::Enabled }` directly now build `AppServerPermissionProfile::External { network: PermissionProfileNetworkPermissions { enabled: true } }.into()` going through the `From` adapter. The `Managed` arm at `:103-124` similarly drops the implicit `network: NetworkSandboxPolicy::Restricted` for an explicit `PermissionProfileNetworkPermissions { enabled: false }` — that's a correctness improvement, not just a rename, because the previous form had network policy hidden inside an enum variant default that future readers would have to grep to understand.

Merge-as-is: the `From` adapter pattern preserves byte-identical behaviour for downstream consumers while letting the protocol crates split cleanly, the tests are exhaustively converted, and the stack has 4/5 + 5/5 successors that depend on this exact shape.

## what I learned
When migrating types between crates, prefer `From`-adapter conversions at the test boundary over direct rewrites — it lets the next PR in the stack delete the old types without churning every test file again, and it surfaces semantic-default cases (the `network: ...` arm) that were previously hidden inside enum variants.
