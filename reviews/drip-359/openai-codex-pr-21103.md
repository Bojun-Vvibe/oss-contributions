# openai/codex PR #21103 — Avoid whole plugin cache replacement on Windows

- Repo: `openai/codex`
- PR: #21103
- Head SHA: `b65f9366eeacf4f1b56e9daf54735b449e404814`
- Author: `zm-oai`
- Updated: 2026-05-04T23:06:55Z
- Verdict: **merge-after-nits**

## What it does

Splits `replace_plugin_root_atomically` into a Unix path and a new Windows path because Windows refuses to rename a directory while any process holds a handle inside it (and the previous code was atomically swapping the entire `<plugin>/` root, which broke if the user's shell `cwd` was anywhere under that root).

- `core-plugins/src/store.rs:263-265` — early-returns to the new `replace_plugin_root_on_windows` when `cfg!(windows)`.
- `core-plugins/src/store.rs:329-380` — `replace_plugin_root_on_windows` only stages and renames at the **version** subdirectory granularity (`<plugin>/<version>/`), not the whole `<plugin>/` root, so a held handle on `<plugin>/<old-version>/...` no longer blocks installing `<plugin>/<new-version>/`.
- `core-plugins/src/store.rs:382-433` — `replace_plugin_version_root_atomically_on_windows` does a backup-and-swap on `<plugin>/<version>/` with rollback: on rename failure it tries to restore the backup; on rollback failure it `keep()`s the temp dir and reports both error messages plus the kept-backup path.
- `core-plugins/src/store.rs:435-453` — `remove_stale_plugin_versions_best_effort` walks `<plugin>/` after a successful version install and best-effort-deletes any sibling version directory that isn't the active one; failures are silently ignored (`let _ = fs::remove_dir_all(...)`).
- `core-plugins/src/store_tests.rs:186-253` — two `#[cfg(windows)]` tests that `set_current_dir` *into* `plugin_base_root` to simulate a held handle on the plugin root, then install a same-version (idempotent reuse) and a new-version, asserting the install succeeds and the old version gets cleaned up.

## Strengths

- **Correct diagnosis.** Swapping the whole `<plugin>/` root on Windows is exactly the operation that fails with `ERROR_SHARING_VIOLATION` when the user's shell or another `codex` process has `cwd` under it. Narrowing the swap to `<plugin>/<version>/` matches how the rest of the codex plugin layout is shaped (versions are siblings, not stacked) and the tests demonstrate the failure mode is reproducible by holding `cwd` on the plugin base.
- **Real rollback path.** The double-failure branch in `replace_plugin_version_root_atomically_on_windows` (`Err(rollback_err)`) calls `backup_dir.keep()` so the operator can manually restore from a known on-disk path, rather than dropping the `TempDir` and silently destroying the backup at the worst possible moment. Error message includes both the original `err` and the rollback `rollback_err`.
- **Tests use the right reproducer.** `std::env::set_current_dir(&plugin_base_root)` in both `windows_install_reuses_valid_existing_version_without_renaming_plugin_root` and `windows_install_adds_new_version_without_renaming_plugin_root` is precisely the failure pattern the bug describes; without that the tests would pass on the old code too.

## Nits / concerns

1. **`remove_stale_plugin_versions_best_effort` is silent on failure.** If the user is mid-shell in `<plugin>/<old-version>/`, the `remove_dir_all` will fail and we'll leak the old version directory forever. That's the *correct* behavior (better than panicking), but it should at minimum `tracing::warn!` so the user/operator knows the cache is accumulating cruft. As written (`let _ = fs::remove_dir_all(entry.path());` at line 452), a stale-version leak is invisible.

2. **`tempdir_in(parent)` for the backup at `store.rs:399-405` lives next to the active plugin root.** That's fine on the same volume (rename stays atomic) but if the parent directory has restricted ACLs (Program Files, AppData under elevated install) and the running `codex` process doesn't have write on `<parent>`, the backup creation will fail and the install errors out before any swap. Worth a comment that the Windows path requires the same `<parent>` write permission as the existing Unix path.

3. **No test for the "version directory exists, reinstall same version" reuse path on Windows.** The first new test (`windows_install_reuses_valid_existing_version_without_renaming_plugin_root`) installs version 1.0.0, sets `cwd` to the base root, then calls `install` again with the same source — but the existing-cache fast path likely short-circuits before reaching `replace_plugin_root_on_windows` at all. The test passes either way, which means the `if !target_version_root.exists()` early-return at line 384 is only exercised by the v1→v2 test. Worth either documenting that or adding an explicit "different content, same version" test.

4. **`#[cfg(windows)]` tests can't be exercised in CI macOS/Linux runners.** The existing CI matrix presumably has a Windows job; if not, this entire fix is reviewed by inspection only. Worth confirming Windows CI is green on this branch.

## Verdict
**merge-after-nits** — accurate diagnosis, correct narrowing of the swap granularity, and the rollback path is the right shape. Add a `tracing::warn!` to the silent-stale-cleanup branch and confirm Windows CI coverage; the rest is fine.
