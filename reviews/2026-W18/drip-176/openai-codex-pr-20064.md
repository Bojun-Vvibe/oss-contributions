# openai/codex#20064 — Include auto-review rollout in feedback uploads

- PR: https://github.com/openai/codex/pull/20064
- Head SHA: `42e9a1fb`
- Author: won-openai (Won Park)
- Base: `main`
- Size: +115 / -? across multiple Rust crates

## Summary

When a user runs `/feedback` and consents to upload logs, this PR includes
the *live* auto-review (Guardian) trunk rollout as an additional attachment,
named `auto-review-rollout-<parent-thread-id>.jsonl` so it's distinguishable
from the parent rollout. Surfaces the new attachment name in the TUI consent
popup. Scope is explicitly bounded: covers the cached live trunk only — no
durable historical lookup, no persisted rollout for ephemeral parallel
review forks (both deferred to follow-ups).

## Notable design choices

- `feedback/src/lib.rs:338+` introduces a new `FeedbackAttachmentPath`
  struct with `path: PathBuf` and `attachment_filename_override:
  Option<String>` instead of widening the existing `Vec<PathBuf>` API.
  Call sites in `codex_message_processor.rs:7737-7745` migrate cleanly to
  the struct shape and the override is `None` for the legacy paths,
  `Some(auto_review_rollout_filename(conversation_id))` for the new
  Guardian path. This is the right shape — keeps the *transport* path
  separate from the *display* name.
- `core/src/codex_thread.rs:372-379` adds `guardian_trunk_rollout_path()`
  as a thin async accessor that delegates to
  `guardian_review_session.trunk_rollout_path()`. Single-purpose, no
  business logic, easy to mock.
- `core/src/guardian/review_session.rs:262-272` is the load-bearing
  function:
  ```rust
  let trunk = self.state.lock().await.trunk.clone()?;
  trunk.codex.session.ensure_rollout_materialized().await;
  match trunk.codex.session.current_rollout_path().await {
      Ok(path) => path,
      Err(err) => {
          warn!("failed to resolve guardian trunk rollout path: {err}");
          None
      }
  }
  ```
  The `ensure_rollout_materialized().await` between unlocking the trunk
  and resolving the path is the right ordering — guardian sessions can be
  in-memory before they have a backing file, so we force the write before
  asking for the path. Error case warns and returns `None`, which means
  "no extra attachment" rather than "feedback upload fails," which is the
  correct fail-open policy here.
- `core/src/session/tests.rs:4663-4694` adds
  `cached_guardian_subagent_exposes_its_rollout_path` which constructs a
  parent + child session, attaches thread persistence to the child via
  `attach_thread_persistence(&mut child_session)`, caches the child as
  the guardian trunk via `cache_for_test`, and asserts the
  `trunk_rollout_path()` returns `Some(child_rollout_path)`. Good — this
  exercises the lock → materialize → resolve chain end-to-end.

## Concerns

1. **`codex_message_processor.rs:7745-7755` `if let conversation_id =
   conversation_id && let Ok(conversation) = ... && let Some(...) && ...`
   chain.** Five `&&` clauses in a single `if let`. Functionally fine
   (Rust let-chains stable since 1.65), but each individual failure mode
   is silently swallowed:
   - missing `conversation_id` → no Guardian attachment, no log
   - `get_thread()` errors → no Guardian attachment, no log
   - `guardian_trunk_rollout_path()` returns `None` → no Guardian
     attachment, no log
   - `seen_attachment_paths.insert()` returns false (path already
     present) → no Guardian attachment, no log

   The third case is "guardian session exists but has no rollout
   materialized yet" which is *expected* during early turns and shouldn't
   warn. The first two are arguably bugs. Consider splitting into named
   intermediate bindings so each failure can be logged distinctly:
   ```rust
   let Some(conversation_id) = conversation_id else { return ... };
   let conversation = match self.thread_manager.get_thread(...).await {
       Ok(c) => c,
       Err(e) => { warn!("..."); return ... }
   };
   ```
2. **`auto_review_rollout_filename(thread_id: ThreadId)` at line 7908.**
   No sanitization on the thread ID. If `ThreadId::Display` ever
   produces a path-traversal-shaped string (`../foo`, `/etc/passwd`),
   the resulting filename `auto-review-rollout-../foo.jsonl` could be
   misinterpreted by the upload server. Cheap defense: assert
   `thread_id.to_string()` matches `[a-zA-Z0-9_-]+` or use
   `thread_id.as_uuid()` if available.
3. **`guardian_review_session.trunk_rollout_path()` holds the state lock
   only for `.trunk.clone()?`.** Then drops the lock before calling
   `ensure_rollout_materialized()`. If two `/feedback` calls race, both
   can see the same `trunk` snapshot, both call materialize, both resolve
   the same path — fine, idempotent. But if the trunk *changes* between
   unlock and resolve (a new guardian session swaps in), the resolved
   path is stale. Probably acceptable given the bounded scope, but worth
   a comment at `review_session.rs:263` documenting the snapshot
   semantics.
4. **No integration test for the TUI consent popup.** PR body lists
   `feedback_upload_consent_popup_snapshot` and
   `feedback_good_result_consent_popup_includes_connectivity_diagnostics_filename`
   as validated, but neither exercises the new Guardian filename
   appearing in the popup (the snapshot would need updating, which the
   PR claims to do but the diff doesn't show in the visible hunks).
   Confirm the snapshot was updated and not just `cargo insta accept`'d
   to make the snapshot test green.
5. **PR body acknowledges "Known unrelated local failures" in
   `tools::runtimes::tests::maybe_wrap_shell_lc_with_snapshot_keeps_user_proxy_env_when_proxy_inactive`
   and `status::*` snapshot drift.** Reviewer should confirm these are
   genuinely unrelated (check `git log -p` on those test files since
   `main` was last green) — easy way for an unrelated regression to ride
   in on a "known flake" disclaimer.

## Verdict

**merge-after-nits** — well-scoped feature that fills a real gap (Guardian
rollout was invisible to triage), with clean separation between transport
path and display name, and the right fail-open posture. The let-chain
silent-swallow at the message processor (item 1) and thread-ID
sanitization (item 2) are the items most worth addressing before merge.
The follow-ups the PR body lists (durable historical lookup, ephemeral
fork persistence) are correctly deferred — both deserve their own PRs.
