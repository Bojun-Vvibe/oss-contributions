# Review: All-Hands-AI/OpenHands#14127 — Reduce GitHub resolver comment noise

- **PR**: https://github.com/All-Hands-AI/OpenHands/pull/14127
- **State**: OPEN
- **Author**: Lumen-Founder
- **Range**: +433 / −16
- **Head SHA**: `3d3b3fa6de9c38f61d9acaaff714c0db1fc60a91`
- **Verdict**: merge-after-nits
- **Reviewer date**: 2026-04-25

## What the PR does

Closes #13737. Today the GitHub resolver leaves two comments on an issue:
an "I'm on it!" acknowledgement and a separate final summary. This PR
edits the original ack comment in place with the final summary, falling
back to a new comment if the original can't be located. Implementation:

- New module `enterprise/integrations/github/github_comment_utils.py`
  with `build_ack_marker`, `append_ack_marker`, `ensure_conversation_link`,
  `build_final_resolver_comment`, and `iter_recent_paginated_items`.
- `github_manager.py:~410` appends a hidden HTML comment marker
  (`<!-- openhands-ack:{conversation_id} -->`) to the initial ack body.
- `github_v1_callback_processor.py:~135` splits the post path into
  `_post_summary_to_github` (error path) and
  `_post_final_summary_to_github` (success). Both funnel through
  `_post_comment_to_github`, which calls `_edit_existing_issue_ack_comment`
  / `_edit_existing_review_ack_comment` before falling back to
  `create_comment`.
- Tests added in `enterprise/tests/unit/integrations/github/`.

## Strengths

- The marker-based design is the right approach: invisible to humans,
  cheap to grep, scoped per-conversation so a stale unrelated ack from a
  different run isn't accidentally edited.
- `iter_recent_paginated_items` walks pages newest-first and bails early
  via `max_items=200`. For long-running issues with hundreds of comments
  this avoids pulling the whole history. Good defensive cap.
- The exception handler now treats `{403, 404, 410, 422, 423, 429}` as
  non-fatal (was only `410` before). Locked issues (423) and forbidden
  (403) are realistic in enterprise installs; previously a single 423
  would crash the callback.
- Tests include both the "edits existing" path and the "falls back to new
  comment" path; the unit test for `iter_recent_paginated_items` covers
  the early-termination case.

## Concerns

1. **Best-effort ordering, but the marker is appended *after* the
   acknowledgement is posted.** In `start_job` the marker is appended to
   `msg_info` after `base_msg` is computed, but I don't see where
   `msg_info` is actually written to GitHub in this diff slice. If the
   creation site does `issue.create_comment(base_msg)` and then reads
   `msg_info` for logging only, the marker never lands on GitHub and every
   future run does a fallback `create_comment`. Worth a `grep` to confirm
   the post site uses the marker-augmented `msg_info`, and a regression
   test that reads back the comment body and asserts the marker is present.
2. **`iter_recent_paginated_items` raises `ValueError` on missing
   `totalCount`.** PyGithub's `PaginatedList` does expose `totalCount`,
   but it's lazy — accessing it triggers a HEAD-like request. If that
   request 5xxs, the resolver falls all the way out instead of degrading
   to "post a fresh comment". Wrap the `totalCount` access in a try and
   return early on failure, so the editing optimization is best-effort.
3. **Marker collision across reruns of the same conversation.** If the
   resolver retries a conversation (e.g., callback re-delivery), the
   second run will edit the first run's final summary in place. That's
   probably desired — but worth a one-line comment in
   `_edit_existing_issue_ack_comment` so the next maintainer doesn't
   "fix" it by appending a UUID.
4. **HTML comment markers and Markdown rendering.** `<!-- ... -->` is
   safe in GitHub markdown today, but if any downstream renderer (mobile
   notifications, Slack mirroring) strips markdown comments differently,
   the marker could become user-visible. Low risk, but document it.

## Nits

- `_post_summary_to_github` is now a 1-line wrapper; consider deleting it
  and updating `handle_callback_error` to call `_post_comment_to_github`
  directly with `conversation_id=None`.
- Test fixture `FakePaginatedList` is duplicable — push it into a tiny
  shared conftest if other tests grow to need it.

## Recommendation

Merge after confirming concern #1 (marker actually lands on the posted
comment, not just on the `msg_info` log string). The other items are
non-blocking. Nice contained refactor.
