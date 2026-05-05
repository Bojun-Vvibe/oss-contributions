# openai/codex #21251 — chore(app-server-protocol): split up v2.rs

- URL: https://github.com/openai/codex/pull/21251
- Head SHA: `28100c84aa660b63808bcb7c5054ef2f42fe7d7a`
- Diff: +12,014 / -11,953 across 22 files (1 deletion of monolithic `v2.rs`, 21 new files under `protocol/v2/`: `account.rs`, `apps.rs`, `command_exec.rs`, `config.rs`, `device_key.rs`, `feedback.rs`, `fs.rs`, `items.rs`, `mcp.rs`, `mod.rs`, `models.rs`, `notifications.rs`, `permissions.rs`, `plugins.rs`, `process.rs`, `realtime.rs`, `server_requests.rs`, `shared.rs`, `thread.rs`, `thread_data.rs`, `turn.rs`)

## Findings

1. PR description correctly labels this "purely a mechanical refactor" — splitting an 11.9k-line file into 21 resource-grouped modules under `codex-rs/app-server-protocol/src/protocol/v2/`. Net delta is +61 lines, which is the cost of duplicated `use` blocks plus the new `mod.rs` re-export surface (+3,762 lines, since it likely retains shared types, prelude re-exports, and macro invocations that don't fit any single resource).
2. The grouping is logical and consistent with the resource naming used elsewhere in the codex-rs tree (account/apps/permissions/realtime/thread/turn). `shared.rs` (418 lines) and `mod.rs` (3,762 lines) are the two files that need close review — `shared.rs` for type cohesion (everything in there must be used by ≥2 sibling modules, otherwise it should move down), `mod.rs` for whether it accidentally re-exports private items or breaks `pub use` paths that downstream crates depend on.
3. The original `v2.rs:1-200` block visible in the diff shows ~140 individual `use codex_protocol::...` imports. Splitting these correctly across 21 files without orphaning any is the highest-risk part of a mechanical refactor — the only safe verification is `cargo check -p codex-app-server-protocol` plus `cargo check --workspace` on a clean tree. PR body does not state which checks were run; please confirm both pass and that `cargo doc --workspace` doesn't introduce broken intra-doc links (a common casualty of module splits).
4. `#[derive(ExperimentalApi, JsonSchema, ...)]` items: if any `#[serde(rename = ...)]` or `#[schemars(...)]` attributes were attached to the parent module via `#![...]` inner attributes in the original `v2.rs`, those need to be re-applied per new file. Worth grepping `git show` for `#![` in the deleted file to confirm none were lost.
5. Reviewability: 22-file mechanical splits are notoriously hard to review by eye. Recommend the reviewer use `git log --follow` friendliness as a tie-breaker — i.e., `git mv` semantics aren't preserved in a single-commit add+delete, but if the author can split into "1 commit per resource module" the bisectability improves dramatically. Not a blocker for a `chore:` PR, but worth a comment.

## Verdict

**Verdict:** merge-after-nits
