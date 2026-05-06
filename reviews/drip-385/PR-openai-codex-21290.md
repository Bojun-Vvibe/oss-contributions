# openai/codex#21290 — Move file watcher out of core

- **Head SHA**: `d227ef6dffb2aac03b0123a9c51382cfdb17ee1a`
- **Stats**: +55 / -17, 12 files (stacked on #21287)

## Summary

Continuation of the `codex-core` slimming arc — after #21287 moved the skills watcher to `app-server`, the generic `notify`-backed filesystem watcher was the last watcher-specific implementation still living inside `codex-core`. This PR extracts it to a new `codex-file-watcher` crate (re-exporting the same `FileWatcher`/`FileWatcherEvent`/`FileWatcherSubscriber`/`Receiver`/`WatchPath`/`WatchRegistration`/`ThrottledWatchReceiver` surface) so `codex-core` can drop the `notify` dependency and stop being a transitive carrier of filesystem-watching primitives.

## Specific citations

- `codex-rs/file-watcher/Cargo.toml:14-21` (new): minimal dep set — `notify` (workspace), `tokio` with the precise feature set (`["macros", "rt", "sync", "time"]` — no `fs`/`net`, no `rt-multi-thread`, no `signal`), `tracing` with `["log"]`, and dev-deps `pretty_assertions` + `tempfile`. The tokio feature pruning is correct: the watcher only needs `sync` for channels, `time` for throttling, `rt` + `macros` for the test runtime. Worth verifying `ThrottledWatchReceiver` doesn't reach into `tokio::time::Interval` (which needs `time`) — yes it does, so `time` is necessary.
- `codex-rs/Cargo.toml:48,165` (workspace): adds `"file-watcher"` to `members` and `codex-file-watcher = { path = "file-watcher" }` to the alias block. Members list is alphabetical (file-search → file-watcher), the alias block is appended in the existing position. Standard pattern.
- `codex-rs/core/Cargo.toml:86` (deletion of `notify = { workspace = true }`) and `core/src/lib.rs:34,198` (deletion of `pub mod file_watcher` + `pub use file_watcher::FileWatcherEvent`): the `pub use FileWatcherEvent` re-export at `lib.rs:198` is removed without a deprecation alias. Any out-of-tree consumer that imported `codex_core::FileWatcherEvent` (rather than `codex_core::file_watcher::FileWatcherEvent`) now hits a hard compile error. Since codex-rs is workspace-internal this is acceptable, but if any SDK crate or downstream tool uses this re-export, they need a coordinated update — `rg "codex_core::FileWatcherEvent" codex-rs/` would confirm zero internal callers.
- `app-server/src/fs_watch.rs:11-16`, `app-server/src/skills_watcher.rs:9-14`, `app-server/src/thread_state.rs:10`: the three call-site updates rewrite imports `codex_core::file_watcher::X` → `codex_file_watcher::X` mechanically. All five symbols (`FileWatcher`, `FileWatcherSubscriber`, `Receiver`, `ThrottledWatchReceiver`, `WatchPath`, `WatchRegistration`) are re-exported from the new crate root since the file rename is `core/src/file_watcher.rs` → `file-watcher/src/lib.rs` (verbatim, similarity 100%).
- `codex-rs/file-watcher/BUILD.bazel:1-6` (new): mirrors the `codex_rust_crate(name=..., crate_name=...)` pattern from sibling crates. The `crate_name = "codex_file_watcher"` (underscores) vs `name = "file-watcher"` (hyphens) split is the standard Bazel-Rust idiom.
- `Cargo.lock` deltas at `:1873`/`:2488`/`:2846`: confirms `codex-core` no longer pulls `notify` (line 17 deletion) and that `codex-file-watcher` is the new owner (`:2849` adds the crate with `notify`/`pretty_assertions`/`tempfile`/`tokio`/`tracing`). Lockfile diff is consistent with the manifest changes — no orphan dependency drift.

## Verdict

**merge-after-nits**

## Rationale

Textbook crate extraction — file rename via git's similarity detection (100% match), surgical Cargo.toml edits, callsite import updates limited to `app-server`'s three files, and the dependency footprint of `codex-core` shrinks (one fewer transitive dep). The tokio feature set on the new crate is correctly minimal, and the BUILD.bazel mirrors the existing pattern. Three nits worth raising: (1) the `pub use codex_core::file_watcher::FileWatcherEvent` deletion at `core/src/lib.rs:198` should land with either a one-cycle deprecation re-export (`#[deprecated] pub use codex_file_watcher::FileWatcherEvent`) or an explicit PR-body callout that out-of-tree SDKs need to retarget — a `rg "codex_core::file_watcher\|codex_core::FileWatcherEvent" codex-rs/` should also be in the PR description as evidence of zero internal callers. (2) The new `file-watcher/Cargo.toml` is missing `description`/`repository`/`readme` keys that other extracted crates set; even for workspace-internal crates this matters for `cargo doc` and any future publish. (3) The PR body says "stacked on #21287" but no marker in the commit/PR-tooling encodes that — once #21287 lands, a rebase note in the description would help reviewers track the actual diff. The BUILD.bazel + Cargo.toml + lib.rs trio is correct as-is; the watcher logic itself is verbatim-preserved (similarity=100%) so no behavior change risk.
