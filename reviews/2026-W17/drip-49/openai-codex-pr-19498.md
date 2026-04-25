---
pr: 19498
repo: openai/codex
sha: 460ddfa6afdb4721a0794b4ee1809b3052f1ed63
verdict: merge-as-is
date: 2026-04-25
---

# openai/codex#19498 — Streamline review and feedback handlers

- **URL**: https://github.com/openai/codex/pull/19498
- **Author**: pakrym-oai

## Summary

Final slice of a multi-PR refactor that flattens JSON-RPC handler
shapes in `codex-rs/app-server/src/codex_message_processor.rs`. This
slice covers the remaining review-start, turn-interrupt, fuzzy-search,
feedback-upload, and git-diff handlers. Behavior over the wire is
unchanged — the goal is to convert the existing `match … { Err(e) =>
{ self.outgoing.send_error(...).await; return; } }` ladders into
`async {}.await` blocks that return `Result<Response, JSONRPCErrorError>`
and centralize the error-emit at the request boundary (which was lifted
out in a prior PR, #19484).

## Reviewable points

- `codex_message_processor.rs:5638-5655` deletes the now-unused
  `send_internal_error` and `send_invalid_request_error` helpers. Both
  were only called from inside the handlers being flattened in this
  PR, so removal is safe; ripgrep on the rest of the crate would
  confirm. Good cleanup.

- `start_review` (around line 7009 in the diff) is the cleanest example
  of the pattern: load_thread → review_request_from_target → branch on
  delivery, all chained with `?`. The previous code had three separate
  `match … Err(err) => { send_error; return; }` blocks; the new code
  is one `async {}.await` returning `Result<(), JSONRPCErrorError>` and
  the caller emits the error once. Strictly easier to read and to keep
  exhaustive.

- `turn_interrupt` (line 97 of the diff) follows the same shape. No
  behavioral diff; just deletion of the local error branch.

- `fuzzy_file_search_session_start` is split into a new
  `fuzzy_file_search_session_start_response` helper (line 279) that
  returns `Result<…, JSONRPCErrorError>`, with the public method
  forwarding to it. Same pattern for `fuzzy_file_search_session_update`
  → `…_response` (line 336) and `upload_feedback` → `…_response`
  (line 374). The naming is consistent with prior slices in this
  refactor.

- One minor observation: the new `_response` suffix doesn't read
  perfectly — these are actually "the part that returns a Result so the
  outer wrapper can emit either ack or error". `…_inner` or `…_impl`
  would have matched Rust convention better, but this is bikeshed-tier
  and consistency with the earlier PRs in the series outweighs it.

- Net diff is **+123/-209**, all in one file. That's a real subtraction
  of duplicated error-handling, not a wash.

## Rationale

This is the kind of tail-end refactor that earns its keep: same
public protocol, fewer lines, fewer ways to forget to emit an error.
No new tests are needed because the behavior is unchanged and the
existing handler tests already cover the success/error pairs. Approve.

## What I learned

When a JSON-RPC dispatch layer has been built up handler-by-handler
with local error-emit branches, the cleanest mechanical refactor is:
(1) lift the error-emit to the request boundary in one PR, (2) flatten
each handler into `Result<…, ErrorObj>` in subsequent PRs, batched by
related domain. This PR is the last batch in that sequence and
demonstrates the pattern paid off — the diff is monotonic deletion.
