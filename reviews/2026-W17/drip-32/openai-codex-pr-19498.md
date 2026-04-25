# openai/codex#19498 — Streamline review and feedback handlers

- PR: https://github.com/openai/codex/pull/19498
- Head SHA: `00c524bf5043c37ee09dacae90c5ecd7cd366832`
- Base SHA: `9de37de3c1cf6157c36f653a3317433056b55a80`
- Author: pakrym-oai
- State: DRAFT (as of 2026-04-25T03:09Z)
- Diff: +123 / −191 (one file: `codex-rs/app-server/src/codex_message_processor.rs`)

## Summary

Final slice of a multi-PR sweep that lifts JSON-RPC error dispatch out
of leaf handler bodies in the app-server and onto the request boundary.
This one targets the review (`start_inline_review` / `start_detached_review`),
turn-interrupt, fuzzy-search, feedback-upload, and git-diff handlers in
`codex_message_processor.rs` (the same file that also hosts most of the
other handler entry points). The protocol surface is unchanged — what
changes is that each handler now builds a single `Result<…, JSONRPCErrorError>`
inside an `async { … }.await` block and hands it to a shared
`send_optional_result` / `send_result` helper, instead of inlining
`self.outgoing.send_error(request_id, err).await; return;` at every fallible
step.

## Diff highlights

- `codex_message_processor.rs:7003-7060` — `start_review` collapses three
  sequential `match … { Err => send_error; return; }` ladders (load_thread,
  `review_request_from_target`, the inline/detached dispatch) into one
  `async` block returning `Ok::<_, JSONRPCErrorError>(None::<ReviewStartResponse>)`,
  then a single `self.send_optional_result(request_id, result).await`. The
  `thread_id.clone()` on the inline branch goes away because the closure
  owns `thread_id` for the whole result computation.
- `codex_message_processor.rs:~7080+` — `turn_interrupt` migrates the
  `expected active turn id … but found …` validation from a stringly-typed
  `Err(format!(…))` arm-then-`send_error` to a typed
  `return Err(invalid_request(format!(…)))` inside the new async block. This
  is a real semantic improvement: the error now carries the
  `INVALID_REQUEST_ERROR_CODE` JSON-RPC code instead of being silently
  whatever the local `send_error` path was constructing.
- The startup-interrupt branch (`is_startup_interrupt = turn_id.is_empty()`)
  still threads through unchanged — good, that behavior is the load-bearing
  case for the v2 startup handshake.

## Concerns / Nits

1. **`None::<ReviewStartResponse>` annotation** is required because the
   `async` block has no other site that names the response type; once this
   ships, a follow-up could push the type into the `send_optional_result`
   signature and drop the turbofish.
2. **Test coverage**: PR description cites
   `cargo test -p codex-app-server --test all v2::review -- --test-threads=1`.
   Worth confirming the turn-interrupt mismatched-id branch (the
   `format!("expected active turn id {turn_id} but found {}", active_turn.id)`
   path) has a regression test that asserts the error code is now
   `INVALID_REQUEST_ERROR_CODE`, not just that an error fires — otherwise the
   semantic upgrade is invisible to clients.
3. This is the *fifth* PR in the sweep (paired with #19484 / #19490–#19497).
   Reviewers should land them in order or rebase carefully; touching the same
   ~140-line region of `codex_message_processor.rs` in five overlapping PRs
   is going to produce ugly conflicts otherwise.
4. Draft status — fine to give early feedback but probably hold approval
   until the sibling PRs in the chain are also marked ready.

## Verdict

**merge-after-nits** — the refactor is mechanical, the protocol behaviour
is preserved, and the JSON-RPC code upgrade on the turn-interrupt mismatch
path is a quiet bonus. Hold for the chain to land in order and for one
test confirming the new error code on the mismatch branch.
