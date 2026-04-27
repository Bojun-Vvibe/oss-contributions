# openai/codex#19717 — fix: fall back to git metadata for HEAD commit hash

- **Author**: bolinfest
- **Size**: +124 / -10 (codex-rs/git-utils/src/info.rs + tests)

## Summary

`get_head_commit_hash()` only trusted `git rev-parse HEAD` for resolving the current commit SHA used in `x-codex-turn-metadata`. On Windows CI, `responses_stream_includes_turn_metadata_header_for_git_workspace_e2e` was flaking because the CLI lookup occasionally failed (process spawn flake, antivirus interference, etc.) even though the `.git/HEAD` and refs on disk were perfectly readable.

Fix: keep the `git rev-parse HEAD` fast path (so common-case behavior is unchanged), but on failure, fall back to a `gix::discover(...).head_id()` lookup that reads `.git/` metadata directly. Handles standard repos, packed-refs, and worktree `commondir` layouts.

## Diff inspection

```rust
// :147-167 — restructured get_head_commit_hash
pub async fn get_head_commit_hash(cwd: &Path) -> Option<GitSha> {
    if let Some(output) = run_git_command_with_timeout(&["rev-parse", "HEAD"], cwd).await
        && output.status.success()
        && let Ok(stdout) = String::from_utf8(output.stdout)
    {
        let hash = stdout.trim();
        if !hash.is_empty() {
            return Some(GitSha::new(hash));
        }
    }
    // Fallback to gix metadata read
    get_head_commit_hash_from_gix(cwd)
}

fn get_head_commit_hash_from_gix(cwd: &Path) -> Option<GitSha> {
    let base = if cwd.is_dir() { cwd } else { cwd.parent()? };
    let head = gix::discover(base).ok()?.head_id().ok()?.detach();
    Some(GitSha::new(&head.to_string()))
}
```

Uses Rust 2024 `let-chains` syntax — same MSRV note as #19706.

The `cwd.is_dir() ? cwd : cwd.parent()?` shim at the top of `get_head_commit_hash_from_gix` handles the case where `cwd` was passed as a file path (e.g. an open file in the editor). `gix::discover` walks upward from the provided base looking for `.git/`.

Test coverage at `:737-840`:
- `create_repo_with_commit` helper sets `GIT_CONFIG_GLOBAL=<empty>` + `GIT_CONFIG_NOSYSTEM=1` so the test doesn't pollute or read user config — disciplined hermetic-test setup.
- `get_head_commit_hash_from_gix_reads_packed_refs` — runs `git pack-refs --all` then asserts gix recovers the SHA.
- `get_head_commit_hash_from_gix_reads_worktree_head` — uses `git worktree add ... -b feature-x` then asserts the worktree's HEAD resolves correctly through the `commondir` indirection.

Both tests assert exact SHA equality with the SHA captured from `git rev-parse HEAD` in the same hermetic environment.

## Strengths

- Fast path unchanged — common-case `git rev-parse` still wins, so no latency regression.
- Fallback uses `gix` (already a transitive dep, presumably) — pure-Rust, no shell-out, immune to the spawn flake that triggered this fix.
- Three layout cases all handled by `gix::discover().head_id()`:
  - loose ref (`.git/refs/heads/main` exists)
  - packed-refs (`.git/packed-refs` consolidated)
  - worktree (`.git/worktrees/<name>/HEAD` + `commondir` pointer)
- Hermetic test environment with `GIT_CONFIG_GLOBAL` + `GIT_CONFIG_NOSYSTEM` is the right way to test git-utils — won't leak user config into CI.
- Pretty assertions (`pretty_assertions::assert_eq`) for diff-friendly failure output.
- `cwd` file-vs-dir guard handles a real edge case (calling site might pass a file path).
- Detailed comment justifying why `gix` is fallback-only and not the primary path: "the common case for the other turn-metadata probes" — preserves consistency.

## Concerns

1. **No test for the loose-ref (non-packed) path.** PR body claims "loose refs, `packed-refs`, and worktree `commondir` layouts" but only the latter two are tested. The default initial-commit case is loose, so technically `create_repo_with_commit` exercises it implicitly — but a third explicit test would document the matrix completely.
2. **Detached HEAD case.** `gix::discover().head_id()` should still resolve a detached HEAD SHA correctly, but no test pins it. Worth one more test to cover `git checkout <sha>` then verify resolution.
3. **`unborn HEAD` (fresh repo, no commits).** Both `git rev-parse HEAD` and `gix::head_id()` should return `None` (the latter as `Err(...)` mapped via `.ok()?`), but no test pins the unborn case. Worth one more.
4. **`GitSha::new(&head.to_string())` round-trips through String.** `gix::ObjectId` has `to_hex()` and `Display` impls — minor perf nit, but `format!("{head}")` or `head.to_hex().to_string()` would skip a debug-format allocation. Not a blocker.
5. **`run_git_command_with_timeout` returning `None` already covers timeout.** The `.await?` early-return path is "no command output at all" — but a timeout-then-disk-fallback is also valid behavior. The new `if let Some(output) = ... await` (without `?`) explicitly drops the error and falls through to gix, which is the desired semantic. Worth a one-line comment confirming "we deliberately don't `?` here because we want the gix fallback on timeout too."
6. **Submodule case.** `gix::discover` walks upward — what does it do inside a submodule whose `.git` is a `gitdir:` pointer file? Should work (gix handles this) but no test pins it.

## Verdict

**merge-as-is** — the fix is surgical, fast path is unchanged, fallback is correct for the three documented layouts, hermetic test setup is exemplary. The detached-HEAD / unborn-HEAD / submodule test additions are completeness improvements, not blockers. The MSRV `let-chains` check is the same one as #19706.
