# Review — openai/codex#20891 — Enforce Windows protected metadata targets

- PR: https://github.com/openai/codex/pull/20891
- Head: `24c064dec138507231a180daee78ba123ad2b125`
- Verdict: **request-changes**

## What the change does

Wires the previously-threaded `protected_metadata_targets` slice into the
Windows sandbox runners (`elevated_impl.rs` and `lib.rs`). New module
`codex-rs/windows-sandbox-rs/src/protected_metadata.rs` introduces
`ProtectedMetadataGuard` with two responsibilities:

1. `deny_paths()` — paths that exist before the run; the caller adds them
   to the WFP deny set.
2. `cleanup_created_monitored_paths()` — iterates `monitored_paths`,
   `symlink_metadata`-checks, then unconditionally `remove_metadata_path`
   anything that appeared. If any cleanup removed a file, exit code is
   forced to `1` when it was `0`.

## Concerns (blocking)

1. **Race window.** Between the `is_err()` existence check and the actual
   removal, the sandboxed process may still hold the handle / may have
   moved the path. Use a single fallible delete + classify by
   `ErrorKind::NotFound`, instead of check-then-delete.
2. **Forced rewrite of exit code.** `if !violations.is_empty() && exit_code
   == 0 { exit_code = 1; }` only flips success → 1, but a process that
   exited with `2` (legitimate domain error) **and** also touched a
   protected metadata path will be reported as `2`, hiding the violation.
   Either always overlay a sentinel exit code (e.g. `0xC0DE`) or surface
   violations on a separate channel — don't multiplex into exit code.
3. **No surfacing of `removed` to the caller.** The function returns
   `Vec<PathBuf>` but both call sites discard the names. Operators
   investigating "why did my command fail" will have nothing to look at.
   At minimum, log via `log_success` / `tracing` with the violated paths.
4. **`prepare_protected_metadata_targets` runs *before* `require_logon_…`**
   in `elevated_impl.rs`. If logon fails the guard is dropped without ever
   running cleanup — fine here because nothing was created yet, but the
   ordering is fragile. Consider taking the guard inside the `Result` of
   the spawn so cleanup is tied to the actual run lifetime.
5. The new module doesn't have a unit test in this PR — at minimum, test
   `cleanup_created_monitored_paths` for: missing path (no-op),
   pre-existing path (skipped), created-during-run path (removed).

Holding pending the race + exit-code handling. Module shape is otherwise
fine.
