# block/goose #9036 — Skip automatic fix which crashes

- URL: https://github.com/block/goose/pull/9036
- Head SHA: `1b16d5aa78682143f28bfd686ecc2d3972255acc`
- Diff: +14 / -159, single file `crates/goose-cli/src/session/builder.rs`

## Findings

1. The PR removes ~145 net lines, primarily by deleting `offer_extension_debugging_help` (visible in the diff starting around line 150 of `builder.rs`) — the function that prompted users to spawn a debug session when an extension failed to load. The PR description ("The fix extension path got into an async bit and then crashed") confirms the function was actively crashing in production, and the chosen remediation is to delete the helper rather than fix it. The corresponding `use goose::config::get_all_extensions` import is also removed (it was only used inside the deleted function), which is a clean ancillary cleanup.
2. **This is a feature regression dressed as a fix.** The "offer to debug" flow was a legitimate UX feature: when an extension fails to start, the CLI offered to spawn a hidden debug session with the developer extension loaded so the user could ask for help diagnosing the failure. Deleting it means users now get the raw error message and nothing else. That may be acceptable as an emergency stop-gap, but the PR doesn't acknowledge the UX loss or mention a follow-up to restore the feature with the async bug fixed.
3. **The actual crash root cause is not identified.** The PR title and body just say "got into an async bit and then crashed". The deleted function does several `.await` calls (`config.session_manager.create_session`, `update_provider`, `add_extension`), any of which could be the offender. Without a root-cause analysis, there's no way to know whether the crash will recur in other code paths that share the same broken async pattern (e.g., the regular session-builder flow that also calls `create_session` and `add_extension`). At minimum, link the upstream panic backtrace or issue.
4. **Missing safer alternatives**: between "delete the feature" and "fix the async bug" there are intermediate options — wrap the helper's body in `tokio::task::spawn_blocking` for the cliclack prompt, wrap the whole call in `panic::catch_unwind` / `tokio::spawn` so a panic in the debug-session bootstrap doesn't kill the parent CLI, or feature-gate the call behind an env var. Any of these would preserve the feature for users who want it while protecting the crash path.
5. No tests touched. There almost certainly weren't tests for `offer_extension_debugging_help` (which is why the async bug shipped), and the PR doesn't add a regression test for the new "extension failure prints error and exits" behaviour either.

## Verdict

**Verdict:** needs-discussion
