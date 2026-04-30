# openai/codex#20339 — refactor(linux-sandbox): refactor linux protected metadata runtime

- URL: https://github.com/openai/codex/pull/20339
- Head SHA: `db1b5f04a2e3e27e59ae4d692cf5077707f233f8`
- Size: +1305 / -1266

## Summary

Net-zero structural refactor of `codex-rs/linux-sandbox/src/bwrap.rs` that lifts `ProtectedCreateTarget`, `SyntheticMountTarget`, `SyntheticMountTargetKind`, `FileIdentity`, `should_leave_missing_git_for_parent_repo_discovery`, and `ancestor_has_git_metadata` out of `bwrap.rs` into new sibling modules (`metadata_paths`, `metadata_guard`, `bwrap_runtime`). `bwrap.rs` now `pub(crate) use crate::metadata_paths::{ProtectedCreateTarget, SyntheticMountTarget, SyntheticMountTargetKind}` (`bwrap.rs:14-17`) and continues to expose the same surface to its existing callers. New `bwrap_runtime.rs` (+372 lines) centralizes the bwrap child-process lifecycle including `BWRAP_CHILD_PID: AtomicI32`, `PENDING_FORWARDED_SIGNAL: AtomicI32`, and the `FORWARDED_SIGNALS = [SIGHUP, SIGINT, SIGQUIT, SIGTERM]` set.

## Observations

- `bwrap.rs:14-17` adds `pub(crate) use crate::metadata_paths::{...}` re-exports so existing call sites compile unchanged. This is the right call for a refactor of this size — the module boundary moves but the type identity stays. Verify with `cargo public-api` or equivalent that no `pub` surface (vs `pub(crate)`) accidentally shifts crates.
- Lines `:21-145` of the old file are deleted (the entire `FileIdentity` struct, the two synthetic-mount constructors, the `should_remove_after_bwrap` impl, and the two helper `fn`s for git-discovery). The body should be byte-identical at the new location — the diff for `metadata_paths.rs` / `metadata_guard.rs` isn't visible in the 200-line slice but the deletion side is clean (no behavioral changes embedded in the move). The next reviewer should `cargo expand` both files and `diff` the relevant items to confirm the move was lossless.
- `bwrap_runtime.rs:11-15` introduces `BWRAP_CHILD_PID: AtomicI32::new(0)` and `PENDING_FORWARDED_SIGNAL: AtomicI32::new(0)` as **module-level statics** rather than per-invocation state. Linux sandbox launches are typically one-shot so this is fine in practice, but flag for the next reviewer that these are now process-global — if any test harness ever runs two `run_or_exec_bwrap(...)` invocations in the same process the signal-forwarding state will alias. The prior code (in `bwrap.rs` body) likely had the same property; the move just makes it more visible.
- `FORWARDED_SIGNALS = &[SIGHUP, SIGINT, SIGQUIT, SIGTERM]` at `bwrap_runtime.rs:14-15` matches the conventional set, but `SIGUSR1`/`SIGUSR2` and `SIGWINCH` are intentionally excluded — worth a one-line comment naming the rationale (the bwrap child should survive terminal resize and user-defined signals). Also `SIGPIPE` is not forwarded; if any wrapped command relies on `SIGPIPE` propagation from a parent shell pipeline it will not arrive.
- `should_leave_missing_git_for_parent_repo_discovery` (deleted at `bwrap.rs:557-571`) is the load-bearing helper for git-worktree discovery from inside the sandbox. It should land at `metadata_paths` with byte-identical body and the same `path.symlink_metadata()` + `mount_root.ancestors().skip(1).any(ancestor_has_git_metadata)` predicate. Confirm in the `metadata_paths` portion of the diff (not visible in this slice) that the `gitdir:` text-file fallback at `:587-589` is preserved — that branch handles git worktrees where `.git` is a regular file pointing into the actual git dir, and dropping it silently regresses worktree support.

## Verdict

**merge-after-nits**

## Nits

- Add a one-liner above `FORWARDED_SIGNALS` explaining the exclusion rationale (no SIGUSR1/2, no SIGWINCH, no SIGPIPE).
- Add a doc comment on the two `AtomicI32` statics warning that `run_or_exec_bwrap` is single-shot per process.
- Confirm `cargo public-api` reports zero pub-surface delta on `codex-linux-sandbox`.
