---
pr: https://github.com/openai/codex/pull/19709
sha: 0146950e
diff: +253/-116
state: OPEN
---

## Summary

Fixes the inactive-thread (delegated worker) apply-patch approval flow so the chat history actually shows *what* the worker is asking to write, instead of an empty approval prompt with no diff. Recovers the buffered `FileUpdateChange` list from the thread event store when a `FileChangeRequestApproval` arrives, converts it to the core `FileChange` shape, and renders a patch-preview history cell *outside* the approval overlay so the diff stays visible while the prompt waits for user input.

## Specific observations

- `codex-rs/tui/src/app/thread_events.rs:160-191` — new `file_change_changes(turn_id, item_id)` walks the per-thread buffer in reverse, looking at `ItemStarted` and `ItemCompleted` notifications for a matching `ThreadItem::FileChange { id, changes, .. }`. Falls back to scanning `self.turns` (committed history) when the buffer doesn't have it. Reverse iteration is the right choice — most recent matching emit wins, which handles the case where the same item id is updated. The `turn_id_matches` helper at `:268-270` treats an empty `request_turn_id` as a wildcard, which mirrors how some approval requests arrive without a turn-id pin; reasonable, but worth a comment explaining *why* empty matches everything (otherwise looks like a bug).
- `codex-rs/tui/src/app_server_approval_conversions.rs:46-69` — `file_update_changes_to_core` converts the wire shape (`FileUpdateChange { path, kind, diff }`) into the core `HashMap<PathBuf, FileChange>` shape. The `PatchChangeKind::Update { move_path }` case correctly threads `move_path` into `FileChange::Update { unified_diff, move_path }` — easy to drop on the floor, didn't. Test at `:110-126` pins the `Add` arm; covering `Update` and `Delete` would harden the contract since those have more shape (especially `Update` with `move_path: Some(...)` for renames).
- `codex-rs/tui/src/app/thread_routing.rs:197-205, :275-281` — `thread_file_change_changes` reads under the `store.lock()` and the call site:
  ```rust
  changes: self
      .thread_file_change_changes(thread_id, &params.turn_id, &params.item_id)
      .await
      .map(crate::app_server_approval_conversions::file_update_changes_to_core)
      .unwrap_or_default(),
  ```
  fills the previously-empty `HashMap::new()`. The `unwrap_or_default()` means "no buffered changes" silently produces an empty diff — same visible behavior as before. That's a safe regression posture, but it would be useful to log at debug level when this happens, because in practice it means the buffer was evicted before approval arrived (timing window worth observing).
- `codex-rs/tui/src/app/thread_routing.rs:328-352` — the new `render_inactive_patch_preview` helper inserts a patch-event history cell *only* when (a) the request is `ApprovalRequest::ApplyPatch`, (b) `thread_label.is_some()` (i.e. it's an inactive worker thread, since active-thread approvals already render via the live event stream), and (c) `!changes.is_empty()`. Three guards, each load-bearing — the active-thread path would double-render without the `thread_label.is_some()` check.
- `codex-rs/tui/src/bottom_pane/approval_overlay.rs:632-655` — the overlay header used to embed the `DiffSummary` directly; this PR removes the diff from the overlay and lets the history-cell render do it instead. That's the right architectural call (overlay = "do you approve?", history = "here's what they want to do"), but it does mean a user who dismisses the diff history cell (e.g. by scrolling) loses visibility into *what* the prompt is asking to approve. Worth documenting in the PR description that the overlay no longer shows the diff inline; reviewers expecting the old shape will be surprised.
- New overlay test at `:1556-1585` (`apply_patch_prompt_with_thread_label_omits_command_line`) pins the new behavior — the prompt no longer renders the diff. Good regression catch.
- The integration test at `tui/src/app/tests.rs:2542-2611` (`inactive_thread_file_change_approval_recovers_buffered_changes`) is the most important assertion in the PR: it enqueues an `ItemStarted` with a `FileChange` item, fires a `FileChangeRequestApproval`, then verifies the resulting `ApprovalRequest::ApplyPatch.changes` HashMap contains the buffered file and the resulting history cell renders `"• Added README.md (+1 -0)"` and `"1 +hello"`. End-to-end pin from wire payload through to rendered text — exactly what's needed for a flow that previously silently dropped data.

## Verdict

`merge-after-nits` — solves a real UX gap (inactive worker patch approvals were content-less), the "buffer-then-recover" architecture is a sensible workaround for the fact that delegated approvals can arrive after the originating event has streamed through. Three follow-ups: (1) extend `file_update_changes_to_core` test to cover `Update { move_path: Some(...) }` and `Delete`, (2) add a debug log when the buffer lookup misses (so the timing-window case is observable in the wild), (3) explicit comment on `turn_id_matches`'s empty-string wildcard.

## What I learned

When the protocol emits "the data" and "the request that needs the data" as two independent messages with no causal linkage, the receiver has to do its own correlation by buffering one side and looking it up when the other arrives. The right rendering split for an approval flow is "overlay shows the question, history shows the context" — colocating both in the overlay seems convenient until you realize the overlay disappears the moment the user decides, taking the context with it.
