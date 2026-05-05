# openai/codex #21170 — tools: remove unused experimental `list_dir` tool

- **Head SHA:** `2a3b47f421063afacf167a0de9907ed192b32951`
- **Base:** `main`
- **Author:** jif-oai
- **Size:** +2 / −734 across multiple files (handler, spec, registry, tests, README)
- **Verdict:** `merge-as-is`

## Summary

Pure dead-code removal. The `list_dir` experimental tool ships a
handler (`codex-rs/core/src/tools/handlers/list_dir.rs`, 294 lines), a
spec builder, registry wiring, and a spec-shape test, but no entry in
the model catalog's `experimental_supported_tools` references it. This
PR deletes all of that surface area in one pass.

## What's right

- **Real provability of the deletion.** The PR isn't claiming "we don't
  *think* anyone uses it" — `experimental_supported_tools` in the model
  catalog is the single ground-truth gate that decides whether a tool
  spec is sent to the model, and `list_dir` doesn't appear in any
  current entry. So no model can call this tool today regardless of
  what handler code exists. Deleting the handler can't regress any live
  model behavior.

- **Symmetric removal across the four touch points.** The pre-PR tool
  required (a) the handler at `codex-rs/core/src/tools/handlers/list_dir.rs`
  (deleted, 294 lines), (b) the `ToolKind` variant + registry insertion
  in `codex-rs/core/src/tools/registry.rs`, (c) the spec builder
  `create_list_dir_tool` in `codex-rs/tools/src/`, and (d) the matching
  test `list_dir_tool_matches_expected_spec` in
  `codex-rs/tools/src/utility_tool_tests.rs:6-49`. All four are removed
  in this PR. The README mentions get cleaned up too — leaving stale
  references in markdown is a common review miss, but the PR catches
  them.

- **Test removal is the right move, not a regression.** Deleting
  `list_dir_tool_matches_expected_spec` (the +44/−0 hunk in
  `utility_tool_tests.rs`) is correct because the test pinned the
  spec-builder output of a function that no longer exists. Keeping a
  test for a deleted producer would be dead test code that the next
  maintainer would have to delete anyway.

- **No `pub use` orphans.** The handler file is `mod`-declared in the
  parent module and the deletion removes both the `mod list_dir;`
  declaration and any `pub use ListDirHandler` re-exports — the diff
  doesn't leave dangling `mod` references that would break the build.

- **Read-deny policy code path is not collateral damage.** The deleted
  handler imported `codex_protocol::permissions::ReadDenyMatcher` and
  used the `DENY_READ_POLICY_MESSAGE` constant. `ReadDenyMatcher` is
  still used by other tool handlers (the read-file path, view-image,
  etc.) so removing this caller doesn't orphan the matcher. Worth
  spot-checking that `DENY_READ_POLICY_MESSAGE` isn't a const this file
  was the sole owner of — if it's defined here it should move; if it's
  imported, no action.

## Nits (non-blocking)

- A one-line CHANGELOG / release-note entry under "removed tools" would
  help downstream forks that may have been carrying patches against
  `list_dir.rs` — silent removal of a 300-line file is the kind of
  thing that surprises rebasers. Not blocking because the tool was
  unreachable, but courtesy-flagging it costs nothing.

- The PR body says "delete the `list_dir` handler and its tests from
  `codex-core`" but the diff also removes from `codex-tools` — minor
  body-vs-diff mismatch, not blocking.

- Worth a follow-up grep across `codex-rs/` for any string-literal
  `"list_dir"` references in fixtures, snapshot files, or response
  schemas that wouldn't show up in symbolic dead-code analysis. Cargo
  builds will catch symbol references, but JSON fixtures won't fail
  loud.

## Risks

- **Effectively zero runtime risk** — the deleted code is unreachable
  from any production path because no model advertises the tool.
- **Compile risk:** Cargo will catch any forgotten reference at build
  time. The PR's own CI signal (assuming green) is sufficient proof.
- **Snapshot/fixture risk:** small but worth a follow-up grep as noted
  above.

## Verdict reasoning

`merge-as-is`: pure deletion of provably-unreachable code, symmetric
across all four required touch points (handler, kind, spec, test),
test removal is correct (not a regression — the test's subject is
gone). −734 LoC for zero behavior change is the kind of cleanup that
makes the next refactor cheaper.
