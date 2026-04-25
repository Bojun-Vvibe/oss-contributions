# openai/codex #19526 — [codex] Order codex-mcp items by visibility

- **Repo**: openai/codex
- **PR**: [#19526](https://github.com/openai/codex/pull/19526)
- **Head SHA**: `340dbbc50aae788c42cb1c295cc6989c8fd85dfd`
- **Author**: aibrahim-oai
- **State**: OPEN (+440 / -433)
- **Verdict**: `merge-as-is`

## Context

Direct follow-up to #19524 (the `codex-mcp` surface-trim). Where
#19524 deleted what the crate exposes, this PR re-orders what
remains so each module reads in `pub` → `pub(crate)` → private
order, with the higher-level MCP flows ahead of leaf utilities.
Pure mechanical move — `+440 / -433` is essentially a wash with
the imbalance coming from a few re-grouped blank lines.

## Design

The PR rewrites `codex-rs/codex-mcp/src/lib.rs:1-72` from a
loose `pub(crate) mod ...` block plus an alphabetically-rough
list of `pub use` re-exports into a structured layout:

1. Top of file: `pub use mcp_connection_manager::{MCP_SANDBOX_STATE_META_CAPABILITY,
   McpConnectionManager, McpRuntimeEnvironment, SandboxState, ToolInfo}` — the
   runtime surface external callers care about first
   (`lib.rs:9-13`).
2. Configuration types: `McpConfig`, `ToolPluginProvenance`,
   `CODEX_APPS_MCP_SERVER_NAME` (`lib.rs:15-17`).
3. Apps-cache key helpers, then snapshot/server discovery,
   then auth machinery, with `mcp_permission_prompt_is_auto_approved`
   and `qualified_mcp_tool_name_prefix` near the bottom as
   leaf utilities (`lib.rs:25-65`).
4. `pub(crate) mod ...` declarations moved to the **bottom**
   of the file (`lib.rs:69-71`) — the mod declarations are now
   "implementation detail" framing rather than the file's lede.

The same shape is applied inside the modules. In
`mcp/auth.rs:41-47` the `McpAuthStatusEntry` struct is hoisted
above `oauth_login_support` (it was previously buried at
`auth.rs:117-122`). Moving the type up matches the
"types-then-functions-that-use-them" pattern already used by
the other `mcp/` files, and lets the rest of the file just
`use` the type without forward references.

The `mcp/mod.rs`, `mcp_connection_manager.rs`, and
`mcp_tool_names.rs` rearrangements (visible from the diff
header churn but not the content cap) follow the same recipe.
PR description confirms: "Kept the change mechanical, with no
behavior changes intended."

## Risks

- **Diff readability for downstream rebases.** Any open branch
  that touches `codex-mcp` will get a noisy 3-way merge against
  this PR — every hunk in `lib.rs` shows as a delete + insert.
  Worth landing in a quiet window or coordinating with the
  other `codex-mcp` work in flight (#19498, the
  `Streamline review and feedback handlers` PR, will likely
  conflict). Not a correctness risk, just a coordination one.
- **The blank-line inflation between visibility groups**
  (`lib.rs:13/14`, `21/22`, `30/31`, `45/46`) is intentional
  but adds 3-4 lines that won't survive `rustfmt --check` if
  the project's rustfmt config has any non-default blank-line
  rules. Worth a quick `cargo fmt --check` confirmation in CI
  before merge.
- Because `lib.rs` order is now meaningful, any new `pub use`
  added later needs to land in the right group. A short
  comment header per group (`// runtime`, `// config`,
  `// snapshot`, `// auth`, `// helpers`) would lock that in
  cheaply — without comments, the next contributor will
  alphabetize and the structure decays.

## Suggestions

- (Nit) Add `// runtime`, `// config`, `// snapshot`,
  `// auth`, `// helpers` section comments in `lib.rs` so the
  visibility-grouping intent survives the next contributor.
- (Nit) Move the bottom-of-file `pub(crate) mod` declarations
  back to the top — the convention in most Rust crates is mod
  declarations first so `cargo doc` and rust-analyzer's
  outline view shows the module tree before the re-exports.
  But if the team explicitly prefers this layout, fine to keep.
- Otherwise this is exactly the kind of mechanical hygiene
  that should land without ceremony.

## Verdict reasoning

`merge-as-is`. The PR is a pure ordering pass; behavior
unchanged. The two suggestions above are taste-level and the
team should set the convention they want — neither blocks
merge. Diff is large but every hunk is a move.

## What I learned

The "delete-then-reorder" two-PR split (#19524 + #19526) is a
nice pattern for crate surface cleanups. Doing both in one PR
makes review near-impossible — you can't tell whether a
disappearing item was deleted or moved. Splitting lets the
reviewer see the **delete** PR diff cleanly (every removal is
real), and then the **reorder** PR is a no-op semantically, so
review collapses to "are these still the same set of items?"
A `diff <(git show HEAD~1:lib.rs | grep '^pub use') <(git show HEAD:lib.rs | grep '^pub use')`
should print empty after reorder — that's the regression test
the PR description should claim.
