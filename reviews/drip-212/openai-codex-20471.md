# openai/codex PR #20471 — Stop emitting apply_patch output delta notifications

- Repo: `openai/codex`
- PR: https://github.com/openai/codex/pull/20471
- Head SHA: `e39f3c56a1def13d7593bca3cde875563b45f8c3`
- State: OPEN
- Files: ~10 (protocol schemas + `bespoke_event_handling.rs` + `apply_patch.rs` runtime + tests + sample), +~80/−~150
- Verdict: **merge-after-nits**

## What it does

Removes the producer of `item/fileChange/outputDelta` v2 notifications
end-to-end while keeping the wire schema deprecated-but-defined for
back-compat:

1. **Source-side stop** in `codex-rs/core/src/tools/runtimes/apply_patch.rs:88-100` and
   `:213-215`: deletes `ApplyPatchRuntime::emit_output_delta(...)` plus its
   two call sites (one for stdout, one for stderr) that previously fired
   `EventMsg::ExecCommandOutputDelta` events tagged with the apply_patch
   call_id immediately after `apply_patch` finished.
2. **Routing simplification** in `codex-rs/app-server-protocol/src/protocol/event_mapping.rs:38`
   and `:118-141`: removes the `is_file_change_output: bool` parameter
   from `item_event_to_server_notification(...)` (it had 7 callers, all
   updated). The `ExecCommandOutputDelta` arm now unconditionally maps
   to `CommandExecutionOutputDelta` instead of branching on a
   stateful "did we just see a file change start?" flag.
3. **State-tracking removal** in `codex-rs/app-server/src/bespoke_event_handling.rs:1162-1175`:
   drops the `thread_state.lock().await.turn_summary.file_change_started.contains(&item_id)`
   lookup that was the only consumer of the routing flag, so the
   `file_change_started` set is now effectively dead data on the
   apply_patch hot path (still populated by file-change `item/started`
   handling — see Nit 1).
4. **Schema deprecation** across 5 schema files (json + ts) and the
   `server_notification_definitions!` macro at `common.rs:1400`: each
   `FileChangeOutputDeltaNotification` site gets a `Deprecated legacy
   notification for apply_patch textual output. The server no longer
   emits this notification.` doc-comment. Type is retained for
   serde-symmetry with archived rollouts.
5. **Test deletions** in `codex-rs/app-server/tests/suite/v2/turn_start.rs:2197-2241`
   and `:2978-3036`: the two tests that asserted observed
   `item/fileChange/outputDelta` notifications are loosened to no
   longer wait for or assert that frame; the `output_delta = Some(...)`
   variable and its 9 follow-on assertions are dropped.
6. **New positive test** at `event_mapping.rs:803-825`: asserts that
   `EventMsg::ExecCommandOutputDelta` always maps to
   `CommandExecutionOutputDelta` (no `FileChangeOutputDelta` arm
   anymore).

## What's right

- Right shape for "stop emitting an event": stop at the source
  (`apply_patch.rs::emit_output_delta`), then drain the routing
  parameter that only existed to disambiguate the now-impossible case,
  then mark the wire type deprecated. The downstream cleanup is forced
  by the function-signature change, which is the least error-prone
  way to flush the dead branch.
- Schema is kept (not removed) so old rollouts that recorded
  `FileChangeOutputDeltaNotification` payloads still deserialize. The
  README at `app-server/README.md:1186` is updated honestly: "deprecated
  legacy protocol entry … retained for compatibility but no longer
  emitted by the server."
- The two integration-test loosenings (`turn_start_file_change_approval_v2`
  and `..._accept_for_session_persists_v2`) drop their `output_delta`
  expectations cleanly; they previously interleaved an
  `output_delta.is_some() && completed_file_change.is_some()` two-flag
  loop, the new form just waits on `completed_file_change` which is the
  actual completion signal anyway.

## Nits (request before merge)

1. **Dead state population.** The `thread_state ... .turn_summary.file_change_started`
   set at `bespoke_event_handling.rs` is no longer read on the
   apply_patch routing hot path. If nothing else reads it, the
   `insert(&item_id)` site (where file-change items start) becomes
   write-only memory growth per turn. Either confirm in the PR
   description that another reader exists, or remove the set in this
   PR. (The PR currently only removes the *reader* in the
   `ExecCommandOutputDelta` arm.)
2. **Sample binary still updated.** `thread-manager-sample/src/main.rs:323`
   loses its `/*is_file_change_output*/ false` argument. Good — but
   this means anyone with a downstream fork using
   `item_event_to_server_notification` as a stable API gets a
   compile break. Worth a CHANGELOG line under "Breaking" naming the
   helper.
3. **No deprecation channel for the event itself**, only doc-comments
   on the schema types. `ServerNotification::FileChangeOutputDelta`
   variant remains constructable by callers who hand-build it, but
   nothing in the producer code path will ever emit it. Worth either:
   (a) `#[deprecated(note = "...")]` on the variant so downstream
   consumers get a compile warning, or (b) leaving a comment at the
   `event_mapping.rs::ExecCommandOutputDelta` arm explicitly stating
   "this used to branch on file_change; that distinction was retired
   by PR #20471 because <reason>". The current state has no breadcrumb
   from the post-fix code back to the deletion rationale.
4. The new test
   `exec_command_output_delta_maps_to_command_execution_output_delta`
   at `event_mapping.rs:803` covers the positive path but not the
   anti-regression: that even when a file-change item just started in
   the same turn, the next `ExecCommandOutputDelta` no longer maps to
   `FileChangeOutputDelta`. A second test case carrying the prior
   `is_file_change_output=true` semantic in spirit ("simulate a
   file-change item started in the same turn, observe routing is
   still command-execution") would pin the behavior change against
   accidental re-introduction.
5. README diff at line 1186 says "retained for compatibility but no
   longer emitted by the server." Worth adding "(see PR #20471)" so
   the next archaeology pass has a single anchor.

## Why merge-after-nits

This is a producer-removal at the right boundary (the runtime, not the
serializer), with the routing parameter deletion forcing a clean diff
through 7 call sites, but it leaves a bag of write-only state
(`file_change_started`) and ships no deprecation signal beyond doc
comments. Fix Nit 1 (dead-set removal or proof-of-other-reader) and
Nit 3 (`#[deprecated]` on the variant), the rest are documentation.
