# openai/codex#21174 — Add Windows missing metadata deny sentinel

- **URL:** https://github.com/openai/codex/pull/21174
- **Head SHA:** `6e60556d73a9df88266e5fe17e2add1e5f9d51f2`
- **Stack position:** part 19 of 21
- **Files touched:** 7 (+204 / −27)
  - `codex-rs/windows-sandbox-rs/src/elevated_impl.rs` (+3 / −2)
  - `codex-rs/windows-sandbox-rs/src/lib.rs` (+5 / −2)
  - `codex-rs/windows-sandbox-rs/src/protected_metadata.rs` (+168 / −15)
  - `codex-rs/windows-sandbox-rs/src/setup_main_win.rs` (+11 / −4)
  - `codex-rs/windows-sandbox-rs/src/setup_orchestrator.rs` (+10 / −2)
  - `codex-rs/windows-sandbox-rs/src/unified_exec/backends/elevated.rs` (+4 / −1)
  - `codex-rs/windows-sandbox-rs/src/unified_exec/backends/legacy.rs` (+3 / −1)

## Summary

Adds the third `ProtectedMetadataMode` — `MissingDenySentinel` — to the Windows sandbox enforcement layer. Existing modes were `ExistingDeny` (deny ACL on a path that already exists) and `MissingCreationMonitor` (watch and reactively delete after creation). The new mode materializes an empty placeholder directory for a missing protected metadata name *before* the command runs and adds it to the deny list, so any in-sandbox attempt to create the path fails up-front rather than being cleaned up after the fact.

## Line-level observations

- `protected_metadata.rs:33-35` — module-doc updated to call out the new "materialize as empty deny sentinel" branch alongside the existing reactive-cleanup branch. The doc accurately describes the contract that `MissingDenySentinel` implements.
- `:62-68` — `arm_sentinel_cleanup(&mut self) -> Result<()>` opens a delete-on-close directory handle (`open_delete_on_close_directory`) for each `sentinel_paths` entry and stores the resulting `SentinelHandle` on the guard. The handle keeps the empty directory alive for the command and guarantees deletion when the handle drops, even on forced parent-process termination.
- `:75-89` — `cleanup_created_paths` was renamed from `cleanup_created_monitored_paths` and now sweeps `monitored_paths.iter().chain(sentinel_paths.iter())`. It also calls `self.sentinel_handles.clear()` first to release the delete-on-close handles, which causes Windows to delete the directories as soon as no other process holds them open. Order is correct (drop handles → re-check filesystem).
- `:92-100` — `Drop for ProtectedMetadataGuard` is the safety net for the panic / early-return path: clears `sentinel_handles` then `let _ = remove_metadata_path(path)` for each sentinel. The `let _ =` swallow is correct because Drop cannot return Result; failure would mean the sentinel is left behind, which the next sandbox invocation would notice via `existing_metadata_path()`.
- `:111-124` — `ProtectedMetadataRuntime::finish` was refactored from "fail-fast on monitor error" to a four-arm match that runs *both* `monitor.finish()` and `guard.cleanup_created_paths()` and combines errors. Important: this guarantees cleanup runs even when the monitor has already failed, which fixes a previously-silent leak window. The combined-error message at `:121-123` includes both errors via `{:#}`, which is helpful in incident triage.
- `:130-145` — `SentinelHandle` is a newtype wrapping `HANDLE` with a `Drop` that calls `CloseHandle` (skipping null and `INVALID_HANDLE_VALUE`) and zeroes the field to make double-drop safe. Standard Windows-resource pattern, correctly applied.
- `:399-435` — `prepare_protected_metadata_targets` now returns `Result<ProtectedMetadataGuard>` (was infallible before). The new `MissingDenySentinel` arm at `:413-422` calls `ensure_missing_deny_sentinel(&target.path)` and merges the resulting deny paths via `protected_metadata_existing_deny_paths`. If existing-deny resolution returns multiple paths (e.g. case-variants on a case-insensitive volume), all of them go on the deny list — defensive against case-collision bypass.
- `:548-565` — `ensure_missing_deny_sentinel(path: &Path) -> Result<bool>`. Returns `Ok(false)` if the path already exists or was created concurrently (`AlreadyExists`), and `Ok(true)` only on the path where this call did the creation. Caller uses the bool to decide whether to claim the path for cleanup (only Codex-created sentinels get cleaned up; pre-existing paths are someone else's responsibility). Correct ownership semantics.
- `:567-585` — `open_delete_on_close_directory` uses `CreateFileW(DELETE, FILE_SHARE_READ|WRITE|DELETE, OPEN_EXISTING, FILE_FLAG_BACKUP_SEMANTICS | FILE_FLAG_DELETE_ON_CLOSE)`. `FILE_FLAG_BACKUP_SEMANTICS` is required to open a directory handle (Win32 API quirk). `FILE_FLAG_DELETE_ON_CLOSE` is the load-bearing flag — guarantees deletion at handle close. Share modes are wide-open by design so other processes inside the sandbox can attempt to access the path (and observe denial).
- `elevated_impl.rs:144-148` and `lib.rs:454-466` — both call sites switch from `prepare_protected_metadata_targets(...)` to `prepare_protected_metadata_targets(...)?` and from immutable to `mut` binding so they can call `arm_sentinel_cleanup()?` after constructing the guard. The arm-sentinel-cleanup call happens *after* the deny ACL paths are inserted into the deny set but *before* the sandbox process is spawned — correct ordering.
- `lib.rs:175-176` — re-export `ensure_missing_deny_sentinel` so the next stack PR (parts 20-21) can wire callers from outside this module.

## Concerns

- `ensure_missing_deny_sentinel` creates the sentinel as a *directory* unconditionally. If the protected metadata name is supposed to be a file (e.g. `.gitignore` style), an empty directory placeholder will satisfy `existing_metadata_path` but may interact oddly with downstream tooling that opens it as a file. Worth confirming whether the targets passed via `MissingDenySentinel` are always directory-shaped, or if a file-vs-dir choice needs to flow through `ProtectedMetadataTarget`.
- `Drop for ProtectedMetadataGuard` at `:92-100` runs `remove_metadata_path` after clearing handles. If `clear()` returns before Windows has actually finalized the delete (delete-on-close is asynchronous in some scenarios), the subsequent `remove_metadata_path` could see an `AccessDenied` from a still-pending delete. The `let _ =` swallows this, which is the right tradeoff in Drop, but a one-line comment explaining the race would help the next reader.
- The four-arm `finish` match at `:114-123` lints as `Err(err), Ok(_) | Ok(_), Err(err)` — readable but slightly opaque. A helper that takes `(monitor_result, cleanup_result)` and returns `Result<Vec<PathBuf>>` would be testable in isolation.

## Verdict

**merge-after-nits**

## Rationale

This PR cleanly slots a third enforcement mode into a stack that's been carefully built up; the type-and-behavior addition is well-isolated. The delete-on-close handle pattern is the right Win32 idiom for "guaranteed cleanup even on parent kill", the four-arm `finish` correctly fixes a previously-silent cleanup-skip on monitor failure, and ownership semantics for "Codex created this vs. someone else created it" are correct. The file-vs-directory question is the most worth resolving before merge; the Drop-race comment and finish-helper extraction are nice-to-haves.
