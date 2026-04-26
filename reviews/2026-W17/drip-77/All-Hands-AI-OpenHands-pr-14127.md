---
pr: 14127
repo: All-Hands-AI/OpenHands
sha: ef76371b4e7c
verdict: merge-after-nits
date: 2026-04-26
---

# All-Hands-AI/OpenHands #14127 — Reduce GitHub resolver comment noise by editing acknowledgement comment

- **URL**: https://github.com/All-Hands-AI/OpenHands/pull/14127
- **Author**: Lumen-Founder
- **Head SHA**: ef76371b4e7c
- **Size**: +436/-16 across 5 files (`enterprise/integrations/github/github_comment_utils.py` (new), `github_manager.py`, `github_v1_callback_processor.py`, two test files)

## Scope

Replaces the previous "post a brand-new summary comment when the resolver finishes" behavior with "find the existing acknowledgement comment and edit it in place". Goal: cut comment-thread noise on GitHub issues that the OpenHands resolver agent works on.

Mechanism:

- **`build_ack_marker(conversation_id) → "<!-- openhands-ack:{uuid} -->"`** (`github_comment_utils.py:17`): hidden HTML-comment marker embedded in every acknowledgement comment so the final-summary pass can find it.
- **`append_ack_marker(message, conversation_id)`** (`:22`): idempotent append (no-op if marker already present, handles trailing newline).
- **`ensure_conversation_link(message, conversation_id)`** (`:31`): idempotent injection of the conversation tracking link.
- **`build_final_resolver_comment(summary, conversation_id)`** (`:44`): composes the final body = `ensure_conversation_link(summary.strip())` + `append_ack_marker`.
- **`iter_recent_paginated_items(paginated_list, max_items=None)`** (`:50`): newest-first iteration over a PyGithub paginated list by walking pages in reverse.

Wiring:

- `github_manager.py:411`: every initial acknowledgement comment now appends the marker via `append_ack_marker(msg_info, conversation_id)` so the final pass can find it later.
- `github_v1_callback_processor.py:81`: callback processor calls `_post_final_summary_to_github(summary, conversation_id)` instead of `_post_summary_to_github(summary)`.
- `github_v1_callback_processor.py:126-227`: the new method searches both issue comments and PR review comments (200 most-recent) for the marker, edits the matched comment in place with the final body, and falls back to creating a new comment if the marker isn't found.

## Specific findings

- **The "edit existing comment" pattern is the right UX choice** for resolver-style bots that can otherwise leave 3-5 comments per issue (ack → progress → progress → summary). GitHub's notification system fires once for the original ack and once for the edit, vs. once per new comment, so noise reduction is real for subscribers. Good direction.

- **HTML-comment marker (`<!-- openhands-ack:{uuid} -->`) is the standard pattern** used by dependabot, renovate, github-actions/stale, etc. Invisible to humans, greppable, idempotent. `build_ack_marker` correctly uses the conversation_id as the discriminator so two concurrent resolver runs on the same issue don't stomp each other's markers. Good.

- **`iter_recent_paginated_items` reads as suspicious at a glance.** It calls `paginated_list.totalCount` (a property that triggers a network round-trip in PyGithub to learn the page count) and then walks pages in reverse. Two concerns:
  - **Cost**: `totalCount` on a large issue's comment list does a HEAD-style request. For an issue with hundreds of comments, this is one extra round-trip per resolver-finish event. Acceptable but worth knowing.
  - **Race**: between reading `totalCount` and walking pages, new comments can be added. A new comment posted during iteration would shift page boundaries, potentially causing duplicate or missed items. For a marker-search use-case, that's tolerable because we're looking for an old comment, not a newly-posted one — but doc-comment that limitation.
  - **Page ordering assumption**: the docstring at `:55-58` says "This assumes the underlying pages are ordered oldest-first". Verify this against PyGithub's `IssueComment` paginated list — if it's actually newest-first (some endpoints are), the iteration is reversed twice and you get the oldest 200 comments instead of the newest. That would silently miss the marker on any active issue.

- **`max_items=200` cap at the call sites** (`:202`, `:224`) is a reasonable safety bound but could miss the marker on an extremely active issue. The acknowledgement marker is *deterministic* (it's posted by openhands itself), so an alternative architecture is: persist the comment ID in the conversation's metadata at ack-time, then look it up by ID at finish-time. That's O(1), no pagination, no race, no scan. Worth filing as a follow-up — the current approach works but is more brittle than necessary.

- **Searching both issue comments AND PR review comments** (`:202` issue comments, `:224` review comments) is correct because a PR resolver run could ack on either surface. But note: review comments have file/line context that issue comments don't; if the original ack was a review comment with line context, the final-summary edit needs to preserve that context (or downgrade gracefully). Reviewer should confirm `comment.edit(body)` on a review comment doesn't strip the path/position.

- **Fallback to "post new comment if marker not found"** (visible in the docstring around `:130`) is correct defense-in-depth: the resolver's final summary always lands somewhere, even if the ack was deleted manually.

- **`build_final_resolver_comment` composition order** at `:46-47`: `ensure_conversation_link` → `append_ack_marker`. So the final body shape is `<summary>\n\nTrack progress [here](...)\n\n<!-- openhands-ack:{uuid} -->`. The marker stays at the very end, which is correct for both grep-friendliness and human readability (HTML comment is invisible, so the visible tail is the tracking link).

- **`ensure_conversation_link` idempotence at `:34`** checks `if link_fragment in message` — that's a substring match, so a previous edit that included the conversation_id in the body (e.g. in a backtick code block) could falsely satisfy the check and prevent the link from being appended. Tighten to a regex that matches the actual `[here](...conversations/{id})` format if you want to be strict; loose substring match is probably fine in practice.

- **`append_ack_marker` idempotence at `:23-24`**: checks for the literal marker substring. Correct (the marker is unique enough). Handles `endswith('\n')` to control double-newline. Good.

- **`hashable summary.strip()` before composition** (`:46`): correctly handles incoming whitespace from upstream summary generation.

- **Tests (`test_github_comment_utils.py`, `test_github_v1_callback_processor.py`)**: visible in the file list. Reviewer should confirm coverage includes: marker idempotence, link idempotence, marker-not-found fallback, multi-page paginated search, and review-comment-vs-issue-comment routing.

- **PR scoped purely to enterprise/integrations/github/**: no leakage into the open-source surface. Reasonable for an enterprise-feature PR.

## Risk

Low-medium. The mechanism is well-trodden (HTML-comment markers are standard practice). Risk concentration is in `iter_recent_paginated_items` — page-ordering assumption, race-with-new-comments, and the `totalCount` round-trip cost. The persistence-of-ack-comment-id alternative would eliminate those risks entirely but is out of scope for this PR.

## Nits

1. Verify PyGithub's `get_issue_comments()` and `get_review_comments()` paginated lists are oldest-first (matches the iteration assumption); if not, the search silently misses recent comments.
2. Confirm `comment.edit(body)` on a review comment preserves path/position context.
3. Consider tightening `ensure_conversation_link`'s substring match to a regex matching the actual `[here](...)` markdown format.
4. File a follow-up: persist ack comment ID in conversation metadata at ack-time so the final-summary pass is O(1) instead of paginated scan.
5. Doc-comment the race window in `iter_recent_paginated_items` (new comments arriving during iteration may shift boundaries).
6. Note the `totalCount` round-trip cost in the function docstring.

## Verdict

**merge-after-nits** — solid noise-reduction approach, idempotent helpers, correct marker scheme, sensible fallback. Page-ordering verification is the gating ask before merge; the rest are doc/follow-up.

## What I learned

The "edit ack comment instead of posting summary" pattern is one of those small UX choices that compounds: every resolver run that hits an issue cuts the comment count from 2 → 1, the notification volume from 2 → 1.5 (edit notifications are softer in most clients), and the "find the bot's status" cognitive load from O(n) scrolling to O(1) (it's always the most recent edit of one specific comment). The implementation cost is the marker scheme + a search-or-fallback pattern, both of which are reusable across any bot that posts to issues. The right architectural endpoint is to persist the comment ID in the conversation's metadata so the search step disappears entirely — but the marker-scan approach is a reasonable starting point and degrades gracefully when the ack comment was deleted manually.
