# openai/codex#20178 — `TUI: Remove core protocol dependency [6/7]`

- URL: https://github.com/openai/codex/pull/20178
- Head SHA: `55eeb5bd0af4`
- Author: etraut-openai
- Size: +0 / -1714 (single deletion of `codex-rs/tui/src/app/app_server_adapter.rs`)
- Stack: 6 of 7

## Summary

Deletes the obsolete `app_server_adapter.rs` bridge between the
app-server event stream and the TUI's direct-core path, now that
PRs 1–5 in the stack have moved every TUI flow onto the app-server
surface directly. The file's own header comment (`tui/src/app/app_server_adapter.rs`
lines 1–11 of the deletion) explicitly calls itself a "temporary
adapter layer ... [that] should shrink and eventually disappear."
This PR is that disappearance.

## Specific references

- `codex-rs/tui/src/app/app_server_adapter.rs` deleted in full
  (1714 lines).
- The file's leading doc comment said: *"As more TUI flows move
  onto the app-server surface directly, this adapter should
  shrink and eventually disappear."* — this PR is the closing
  step of that plan.
- PR description claims `cargo check -p codex-tui` and grep for
  `codex_protocol::protocol` in `codex-tui` are both clean. With
  this stack and #20176, the entire `codex-tui` crate's direct
  dependency on `codex_protocol::protocol` is gone.

## Risks

- The diff is a pure deletion, but the dangerous bit is what
  *imports* this file. If anything outside `codex-rs/tui` pulled
  `app_server_adapter` symbols (e.g. via `pub use`), this breaks
  them. PR description says only the TUI consumed it, but a quick
  `rg "app_server_adapter"` across the workspace would be a
  one-line confirmation worth pasting into the PR.
- Test artifacts that imported via `#[cfg(test)] use ... split_command_string`
  (visible in the file header at line 21 of the deleted file)
  need to live somewhere. The PR doesn't say where they moved.
  If they were unused, that's fine; if they migrated, a sentence
  pointing at PR 5 or PR 7 in the stack would help the reviewer.

## Verdict

`merge-as-is` — assuming the prior PRs in the stack landed and
`cargo check --workspace` is green.

## Suggested nits

- Add a one-line `rg "app_server_adapter"` confirmation in the PR
  body to prove no external consumer remains.
- Optional: in the merge commit message, link to the original
  introduction PR of the adapter so future archaeologists can
  see the lifecycle.
