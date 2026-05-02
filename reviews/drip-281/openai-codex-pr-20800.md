# openai/codex PR #20800 — Show /goal-started threads in resume picker

- Repo: `openai/codex`
- PR: #20800
- Head SHA: `b5c12d60400cb5d4ddb171df5bb72017dfdf2960`
- Author: etraut-openai

## Summary
When `/goal <objective>` is the first interaction on a lazily-created
thread, the rollout file doesn't yet exist and the thread has no
`first_user_message`, so the resume picker can't see it. This PR
materializes the rollout, then seeds a synthetic `/goal <objective>`
user message item if and only if the thread has no prior user history,
so the existing listing/preview path picks it up. Addresses #20792.

## Specific references
- `codex-rs/app-server/src/codex_message_processor/thread_goal_handlers.rs:106-167`
  — new branch guarded by `if let Some(objective) = objective &&
  running_thread.is_some()`. Correctly:
  - Calls `self.thread_store.persist_thread(thread_id)` first to materialize
    the rollout (line 110-117).
  - Reads `metadata.first_user_message` and only seeds when it's `None` or
    whitespace via `is_none_or(|message| message.trim().is_empty())`
    (line 130).
  - Builds a `RolloutItem::EventMsg(EventMsg::UserMessage(UserMessageEvent
    { message: format!("/goal {objective}"), ... }))` and appends + flushes.
- `codex-rs/app-server/src/codex_message_processor.rs:372,384` — adds the
  `UserMessageEvent` import and `StoreAppendThreadItemsParams` import. Both
  used cleanly by the new code path.
- `codex-rs/app-server/tests/suite/v2/thread_list.rs:204-280` — new
  `thread_list_includes_thread_started_with_goal` test enables `[features]
  goals = true`, starts a thread, then sets a goal and verifies the thread
  appears in `thread_list`. Solid end-to-end coverage of the regression.

## Verdict
`merge-after-nits`

## Rationale
Surgical fix that respects the resume-picker contract documented in the
PR body ("a resumable thread has a materialized rollout and a first-user-
message preview"). The "only seed if no prior user history" guard is
the right choice — it keeps existing threads' previews stable.

Nits / questions for the author:
1. The synthetic seeded message includes `images: None, local_images:
   Vec::new(), text_elements: Vec::new()`. Worth confirming downstream
   rollout consumers handle `text_elements: vec![]` as
   indistinguishable from a real empty-text-elements user message — i.e.
   nothing keys off of "this UserMessageEvent originated from `/goal`"
   versus a real `/goal` typed by the user. If they do, this PR
   accidentally papers over that distinction.
2. The metadata read at line 121 is racy with concurrent writers: if
   another path appends a real first user message between
   `persist_thread` and `get_thread`, this code will skip seeding (good)
   and the user-typed message becomes the preview (also good). The
   reverse race — real message arrives between `get_thread` and
   `append_items` — would result in two consecutive user messages in the
   rollout, with the synthetic `/goal` one second. Worth a sentence in
   the comment acknowledging this is acceptable because of the
   `running_thread.is_some()` precondition (the user can't have typed
   anything yet) — but the precondition isn't actually airtight if the
   chat widget could submit ahead of the goal RPC.
3. `format!("/goal {objective}")` does not escape `objective`. If
   `objective` contains a newline, the rollout preview will be the part
   up to the first newline. Probably fine for previews, but worth a
   comment.

No banned strings.
