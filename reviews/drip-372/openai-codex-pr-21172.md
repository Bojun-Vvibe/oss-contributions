# openai/codex#21172 — Add Windows missing metadata monitor runtime

- **URL:** https://github.com/openai/codex/pull/21172
- **Head SHA:** `6df1455723e4254ce7b7ac59a79d60f5daa0a24e`
- **Stack position:** part 17 of 21
- **Files touched:** 2 (+351 / −1)
  - `codex-rs/windows-sandbox-rs/BUILD.bazel` (+1 / −0)
  - `codex-rs/windows-sandbox-rs/src/protected_metadata.rs` (+350 / −1)

## Summary

Adds the Windows runtime object (`ProtectedMetadataRuntime`) that wraps an existing `ProtectedMetadataGuard` with an OS event listener (`MissingCreationMonitor`) over each parent of a missing-protected-metadata path. While a sandboxed command runs, Win32 `FindFirstChangeNotificationW` watches `FILE_NOTIFY_CHANGE_FILE_NAME | DIR_NAME | CREATION` under each parent; on every notification the listener re-enforces by deleting any matching protected-metadata path that the command created. This PR adds the runtime in isolation; subsequent stack PRs wire callers to use it.

## Line-level observations

- `protected_metadata.rs:51-57` — `into_runtime(self)` consumes the guard and returns a `ProtectedMetadataRuntime { guard, monitor }`. Ownership transfer is correct: the runtime owns the guard for the lifetime of the command and the monitor lifetime is tied to the runtime, so a panic between `into_runtime()` and `finish()` will drop both together.
- `:71-78` — `finish()` calls `monitor.finish()` (which stops the OS listeners and returns paths the listener removed) then unions with `cleanup_created_monitored_paths()` from the existing guard (which sweeps anything the listener missed at the close window). De-duped via `unique_paths()`. Belt-and-suspenders is correct here because the listener can race with command exit.
- `:103-119` — `MissingCreationMonitor::start` early-returns a no-op monitor when `paths.is_empty()`, with `stop_event = 0` and empty vecs. The empty-path case avoids creating a `CreateEventW` handle that would never be signaled. Good.
- `:121-127` — `CreateEventW(null, TRUE, FALSE, null)` creates a manual-reset event (the `TRUE` second arg). Manual-reset is correct because every listener thread races on `WaitForMultipleObjects` and they all need to see the stop signal; an auto-reset event would only wake one.
- `:138-152` — `FindFirstChangeNotificationW(parent_wide, FALSE, FILE_NAME | DIR_NAME | CREATION)`. The `FALSE` (`bWatchSubtree`) is intentional and correct: parents are at most one level above the protected metadata path, so subtree watching would over-fire and add cost.
- `:170-176` — listener does `enforce_monitored_paths(&watched_paths, ...)` *before* entering the wait loop. This handles the race where a path was created between `prepare_protected_metadata_targets()` and `into_runtime()`. Important.
- `:184-205` — wait loop branches on `WAIT_OBJECT_0` (change handle), `WAIT_OBJECT_0 + 1` (stop event), `WAIT_FAILED`, and a default-error arm. Each error path calls `record_monitor_error` and breaks; the change-handle path calls `FindNextChangeNotification` to re-arm and breaks if that fails. Cleanly defensive.
- `:218-225` — listener thread is named `codex-protected-metadata-monitor`, which will surface in `procmon` / debugger thread lists. Helpful for incident triage.
- `:228-233` — `FindCloseChangeNotification(change_handle)` is called on both the spawn-failure path (inside `map_err`) and at the end of the listener body. Two-path cleanup is correct.
- `:240-261` — `finish()` calls `stop_listeners()` first then drains the error and removed-path vecs, with `Mutex::lock().map_err(|_| anyhow!("poisoned"))` on each. If a listener panicked while holding the mutex, `finish()` will surface the poison rather than silently dropping data.
- `BUILD.bazel:10` — `unit_test_timeout = "long"`. Justified — these tests spawn real OS threads and exercise `WaitForMultipleObjects(INFINITE)`-shaped waits, so stretched timeouts under loaded CI are realistic.

## Concerns

- The OS listener uses `WaitForMultipleObjects(..., INFINITE)`, so a stuck change-handle (e.g. parent directory unmounted) would wedge the listener thread until `stop_event` fires. `stop_listeners()` does fire it, so this is bounded by `finish()` being called — which the runtime contract guarantees. Worth a one-line comment at `:184` calling out the lifetime invariant explicitly.
- `record_monitor_error` accumulates strings into `Arc<Mutex<Vec<String>>>` with no cap. A pathological notification storm could grow the vec unboundedly across the lifetime of one command. Practical exposure is low (commands are short-lived) but a max-N cap would be defensive.
- The `bWatchSubtree=FALSE` choice depends on `monitored_paths_by_parent` always grouping by direct parent. That helper is in the existing module and not in this diff; reviewers should confirm it does not collapse grandparents.

## Verdict

**merge-after-nits**

## Rationale

Stack PR with a clear isolation boundary and a tight, well-defended Win32 implementation. The listener is correctly armed before the wait loop, cleanup is symmetric, the empty-path case is handled, and the BUILD.bazel timeout bump is justified by the test shape. The two nits (lifetime comment, bounded error vec) are non-blocking but worth addressing before this lands as the foundation for parts 18-21.
