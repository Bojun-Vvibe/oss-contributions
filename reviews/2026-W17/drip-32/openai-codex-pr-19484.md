# openai/codex#19484 — Lift app-server JSON-RPC error handling to request boundary

- PR: https://github.com/openai/codex/pull/19484
- Head SHA: `b772dfdd04b50ebeeda50719fd3be19cfea445bb`
- Base SHA: `8a559e7938bd841d78dc2c442deaa76424faf1d9`
- Author: pakrym-oai
- State: DRAFT
- Diff: +268 / −523 across `codex-rs/app-server/src/error_code.rs`,
  `message_processor.rs`, `command_exec.rs`, `outgoing_message.rs`, plus the
  config / filesystem / device-key / external-agent-config / fs-watch leaves.

## Summary

Foundational PR for the streamlining sweep that #19490–#19498 build on.
It introduces shared JSON-RPC error constructors in
`codex-rs/app-server/src/error_code.rs` (`invalid_request`, `invalid_params`,
`internal_error`) and lifts request-result emission into
`codex-rs/app-server/src/message_processor.rs` so responses and errors are
dispatched at the request boundary instead of inside each leaf handler.
Net diff is heavily deletion-leaning (−523 / +268), almost entirely
duplication removal — every leaf module had its own private
`fn invalid_request(message: String) -> JSONRPCErrorError { … }` cluster,
and those all collapse to imports.

## Diff highlights

- `command_exec.rs:34-36` — switches three `use error_code::INVALID_*` constant
  imports to the new function constructors `internal_error`, `invalid_params`,
  `invalid_request`. The call sites then drop their `.to_string()` calls —
  e.g. `command_exec.rs:158`, `:178`, `:184`, `:249`, `:312`, `:421`, `:635`,
  `:643`, `:665` all go from `invalid_request("…".to_string())` to
  `invalid_request("…")`. Whether the new helpers take `impl Into<String>`
  or `&str + .to_owned()` internally matters for allocation footprint —
  worth confirming the signature is `impl Into<String>` so static-string call
  sites still elide the heap copy.
- `command_exec.rs:683-715` — deletes the trio of private
  `fn invalid_request / invalid_params / internal_error` definitions that
  every leaf had been carrying. Mirror deletions land in the other
  leaf files based on the PR summary.
- The big win is on `command_exec`'s `handle_process_write` and
  `terminal_size_from_protocol` paths where the local validation errors now
  cleanly bubble to the request boundary instead of being constructed and
  immediately wrapped.

## Concerns / Nits

1. **Allocation regression risk**: ~30+ call sites flip from
   `"static literal".to_string()` to `"static literal"`. If the constructor
   signature is `fn invalid_request(s: String)`, the compiler will insert a
   `.to_string()` anyway and the diff is a wash; if it's
   `impl Into<String>` or `Cow<'static, str>`, this is a small win. If
   it's `&'static str`, the dynamic `format!(…)` call sites
   (e.g. `format!("expected active turn id {turn_id} but found {}", …)`
   used in the sibling #19498) won't compile against the helper without a
   second overload — worth verifying the helper signature handles both.
2. **Removal of leaf wrappers** that were "only forwarding to a response
   helper" is mentioned in the description but not visible in the
   `command_exec.rs` portion of the diff — those landings are presumably in
   `message_processor.rs` and the smaller leaf files. Reviewers should spot-
   check that none of those wrappers were doing anything besides dispatch
   (e.g. tracing, metrics) before deletion.
3. **Coordination with #19498 / #19490–#19497**: this is the load-bearing
   prerequisite. Land first, then the rest can rebase cleanly. Until then
   the sibling PRs will keep needing rebases.
4. Verification command in the PR (`cargo test -p codex-app-server --lib
   command_exec::tests`) only covers `command_exec`. The protocol contract
   is best validated end-to-end by running the full
   `cargo test -p codex-app-server --test all` suite, which the chain PRs
   each run on smaller scopes.

## Verdict

**merge-after-nits** — this is the right shape for the codebase and the
deletion ratio speaks for itself. Confirm the helper signature is
`impl Into<String>` (so dynamic `format!` callers stay ergonomic and
static-string callers stay alloc-free) and run the full app-server test
suite once before flipping out of draft.
