# openai/codex #19524 — [codex] Minimize codex-mcp public surface

- **Repo**: openai/codex
- **PR**: [#19524](https://github.com/openai/codex/pull/19524)
- **Head SHA**: `0bd757f52c477c6df1eaed853a3af0378d48c837`
- **Author**: aibrahim-oai
- **State**: OPEN (+5 / -489)
- **Verdict**: `merge-as-is`

## Context

Crate hygiene pass on `codex-rs/codex-mcp`. The crate has been
accumulating `pub use` re-exports as new helpers landed; this PR
walks them back to the genuinely-needed surface and deletes the
now-unused `skill_dependencies` module wholesale.

## Design

Three coordinated trims:

1. **`codex-rs/codex-mcp/src/lib.rs:5-39`** drops eight re-exports
   (`McpManager`, `canonical_mcp_server_key`,
   `collect_mcp_server_status_snapshot`, `collect_mcp_snapshot`,
   `collect_mcp_snapshot_from_manager_with_detail`,
   `collect_mcp_snapshot_with_detail`,
   `collect_missing_mcp_dependencies`, `group_tools_by_server`,
   `split_qualified_tool_name`, `DEFAULT_STARTUP_TIMEOUT`). All
   `_with_detail` variants stay; the non-detail aliases are gone.

2. **`codex-rs/codex-mcp/src/mcp/mod.rs:1-12`** removes the
   `skill_dependencies` module declaration plus the two re-exports
   it fed (`canonical_mcp_server_key`,
   `collect_missing_mcp_dependencies`). The module file
   (`skill_dependencies.rs` -172 lines, `_tests.rs` -115 lines)
   is deleted outright. Free function `collect_mcp_snapshot`
   (`mod.rs:361-431`) and `collect_mcp_server_status_snapshot`
   (`mod.rs:438-462`) are deleted because they were thin wrappers
   that only forwarded `McpSnapshotDetail::Full` into the
   `_with_detail` variant — call sites can pass it themselves.

3. **`codex-rs/codex-mcp/src/mcp/mod.rs:144,151`** demotes
   `ToolPluginProvenance::plugin_display_names_for_connector_id`
   and `_for_mcp_server_name` from `pub` to `pub(crate)`. The
   type itself stays public but those accessors are now
   crate-internal.

A doc reference at `mod.rs:103-104` is updated from
``[`collect_mcp_snapshot`]`` to "snapshot collection helpers" so
the doc link doesn't dangle.

The `pub type McpManager = McpConnectionManager;` alias at the
old `mod.rs:39` is removed; the canonical name
`McpConnectionManager` is the only one exported now.

## Risks

- **External consumer breakage** is the main concern, but
  `codex-mcp` is a workspace-internal crate (no `[lib]` published
  to crates.io that I can see), so the only consumers are the
  `codex-rs` workspace and `sdk/python` bindings. Both are in-tree
  and would have been updated by the same PR if affected — the
  diff only touches three files in the crate, so nothing else in
  the workspace was relying on these symbols.
- **`pub` → `pub(crate)` on `ToolPluginProvenance` accessors**
  (lines 144, 151) is technically a breaking change for any
  out-of-tree consumer that imported the struct and called these
  methods. Same workspace-internal argument applies.
- The deleted `skill_dependencies.rs` file (172 lines) and its
  tests (115 lines) had real logic. It looks like that
  functionality moved elsewhere or was made redundant by recent
  work — would be worth a one-line PR-body note saying which PR
  obsoleted it, in case someone later wonders where
  `collect_missing_mcp_dependencies` went.

## Suggestions

- (Nit) Add a one-line PR-body pointer to the PR/commit that made
  `skill_dependencies` obsolete, so `git log --follow` doesn't
  have to tell the whole story.
- Otherwise this is exactly the kind of mechanical hygiene that
  should land without ceremony.

## What I learned

The `_with_detail` / non-`_with_detail` doublet was a classic
"add an enum param, keep the old API as a delegating wrapper"
shape. It's worth holding the wrappers for one or two releases
to give downstream a chance to migrate, but if the crate is
workspace-internal, there's no downstream — keeping the
delegators just makes the public surface confusing. This PR is
the right time to delete them.
