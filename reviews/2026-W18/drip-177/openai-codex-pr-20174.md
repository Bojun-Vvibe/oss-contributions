# openai/codex#20174 — TUI: Remove core protocol dependency [3/7]

- **Repo:** openai/codex
- **PR:** [#20174](https://github.com/openai/codex/pull/20174)
- **Head SHA:** `159c23c148acbeee7db86421ada0676af47c0e85`
- **Author:** etraut-openai
- **Size:** +165 / -152 across 19 files

## Summary

Part 3 of a 7-PR stack stripping direct `codex_protocol::protocol` usage
from `codex-tui`. This step changes `AppEvent::CodexOp` and
`AppEvent::SubmitThreadOp` to carry `AppCommand` directly instead of
bouncing through core `Op` values, and lifts `ApproveGuardianDeniedAction`
from `Other(Op::ApproveGuardianDeniedAction { event })` into a first-class
`AppCommand` variant.

## What's actually going on

The mechanical change at `codex-rs/tui/src/app/event_dispatch.rs:316-323`
drops the `.into()` conversions that were translating `AppCommand` → `Op`
on the dispatch boundary:

```rust
AppEvent::CodexOp(op) => {
-    self.submit_active_thread_op(app_server, op.into()).await?;
+    self.submit_active_thread_op(app_server, op).await?;
}
...
AppEvent::SubmitThreadOp { thread_id, op } => {
-    self.submit_thread_op(app_server, thread_id, op.into()).await?;
+    self.submit_thread_op(app_server, thread_id, op).await?;
}
```

The downstream `submit_active_thread_op` / `submit_thread_op` signatures
must therefore now take `AppCommand` instead of `Op` — which means every
in-tree caller had to flip its construction site from
`AppCommand::override_turn_context(...).into_core()` to plain
`AppCommand::override_turn_context(...)`. The diff at
`codex-rs/tui/src/app/config_persistence.rs:351-368` and the two sites in
`event_dispatch.rs:988` / `event_dispatch.rs:1013` all show this
shape — drop the trailing `.into_core()` / `.into()`.

The `ApproveGuardianDeniedAction` lift is the substantively-new piece. At
`codex-rs/tui/src/app_command.rs:107-109` it adds an `AppCommand` variant
and at `:191-193` an `AppCommandView` variant, then at `:457-459` extends
`into_core()` to translate back to `Op::ApproveGuardianDeniedAction`. The
matching dispatcher change at `codex-rs/tui/src/app/thread_routing.rs:707-712`:

```rust
AppCommandView::ApproveGuardianDeniedAction { event }
| AppCommandView::Other(Op::ApproveGuardianDeniedAction { event }) => {
    app_server
        .thread_approve_guardian_denied_action(thread_id, event)
        .await?;
```

handles *both* the new first-class variant and the legacy `Other(Op::...)`
shape, which is the right migration posture — call sites can be updated
incrementally without breaking thread routing mid-stack.

## Specific line refs

- `codex-rs/tui/src/app/event_dispatch.rs:316` — `.into()` removed from `CodexOp` arm.
- `codex-rs/tui/src/app/event_dispatch.rs:323` — `.into()` removed from `SubmitThreadOp` arm.
- `codex-rs/tui/src/app/event_dispatch.rs:988-991` — caller updated from
  `.into()` to direct `AppCommand` construction (override_turn_context world-writable warning path).
- `codex-rs/tui/src/app/event_dispatch.rs:1013-1016` — same pattern, ApprovalPolicy update path.
- `codex-rs/tui/src/app/config_persistence.rs:351-368` — `.into_core()` dropped from
  the Windows-sandbox `override_turn_context` send path; the `#[cfg(target_os = "windows")]`
  gate still guards the whole block.
- `codex-rs/tui/src/app_command.rs:107-109` — new `ApproveGuardianDeniedAction` variant.
- `codex-rs/tui/src/app_command.rs:191-193` — corresponding `AppCommandView` borrow variant.
- `codex-rs/tui/src/app_command.rs:457-459` — `into_core()` translation back to `Op`.
- `codex-rs/tui/src/app/thread_routing.rs:707-712` — dual-pattern match accepts both old `Other(Op::...)` and new variant.
- `codex-rs/tui/src/app/tests.rs:444-451` — test patterns flipped from `Op::UserTurn` to `AppCommand::UserTurn`.

## Reasoning

This is a clean intermediate refactor PR in a well-staged 7-step removal of
core protocol coupling. The dual-pattern match at `thread_routing.rs:707-712`
demonstrates the author understands they can't break the next 4 PRs in the
stack — they're keeping the legacy `Other(Op::ApproveGuardianDeniedAction)`
path live so steps 4-7 can migrate other call sites at their own pace
without a flag day. That's the right discipline for a stack of this size.

The only risk specific to step 3 is the `into_core()` removals in the
`#[cfg(target_os = "windows")]` block at `config_persistence.rs:351-368`.
Windows-only code paths frequently break across refactors that compile fine
on macOS/Linux because nobody runs the Windows test job locally. Worth
confirming CI's Windows job is green on this PR before merging — the
`cfg(target_os = "windows")` gate means the `submit_active_thread_op` call
on that path won't be exercised by default.

The test in `app/tests.rs:441-450` is doing the right job — it asserts the
pattern at the boundary, so if step 4 (which presumably removes the
`Other(Op::...)` legacy arm) reintroduces an accidental `Op` shape on the
event channel, this test will catch it. Good defense.

A couple of observations:

1. **`AppCommandView` borrow variant.** Adding a borrow variant alongside
   the owned variant (`AppCommand::ApproveGuardianDeniedAction { event }`
   vs `AppCommandView::ApproveGuardianDeniedAction { event: &'a ... }`) is
   the right shape — readers can pattern-match without forcing an `Op`
   clone. But this is the *third* place that pattern lives in `app_command.rs`
   alongside `Review` and `Other` variants; if the count keeps growing across
   steps 4-7, consider a `#[derive(...)]` or a macro to keep the
   `AppCommand` ↔ `AppCommandView` ↔ `into_core` triple in sync. Out of scope
   for this PR.
2. **No CHANGELOG / migration note.** The PR body promises "each layer
   reviewable and shippable" — for a 7-PR stack landing on `main`, a one-line
   note in `CHANGELOG.md` or a tracking issue with checkboxes for each step
   would help downstream consumers understand the staged migration.

## Verdict

**merge-after-nits** — confirm the Windows CI job is green before merge
(the `cfg(target_os = "windows")` block at `config_persistence.rs:351-368`
isn't covered by macOS/Linux test runs, so a silent type-mismatch on the
Windows-only path wouldn't be caught locally), add a one-line
`CHANGELOG.md` note tracking the 7-PR stack so consumers can audit the
migration's progress, and either link or inline a TODO at
`thread_routing.rs:707` noting which step in the stack will remove the
legacy `Other(Op::ApproveGuardianDeniedAction)` arm so it doesn't become
permanent dead code. Body of the change is correct and the dual-pattern
matching is the right discipline for an in-flight stack.
