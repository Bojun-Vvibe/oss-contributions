# openai/codex#20173 — TUI: Remove core protocol dependency [2/7]

- **Repo:** openai/codex
- **PR:** #20173
- **Head SHA:** 6cc2aa221bc1f776ece9cef0e5ea191e1123b1a5
- **Base:** etraut/tui-cleanup-1 (PR-stack base, not main)
- **Author:** etraut-openai (maintainer)

## Summary

Step 2 of a 7-PR stack stripping `codex_protocol::protocol::Op` from `codex-tui`. Replaces the opaque `AppCommand(Op)` newtype wrapper at `codex-rs/tui/src/app_command.rs:28` with an explicit `enum AppCommand { Interrupt, CleanBackgroundTerminals, RealtimeConversationStart(...), UserTurn { ... }, OverrideTurnContext { ... }, ExecApproval { ... }, PatchApproval { ... }, ... }` carrying owned variants for every command the TUI submits. Preserves an `into_core()` shim (existing call sites untouched) so the app/thread submission boundary doesn't move yet — this PR is a pure shape refactor that unblocks step 3+.

## File:line references

- `codex-rs/tui/src/app_command.rs:28-110` — new `AppCommand` enum with ~20 owned variants replacing `pub(crate) struct AppCommand(Op)`
- `codex-rs/tui/src/app_command.rs:9` — `#[allow(clippy::large_enum_variant)]` added (`UserTurn` carries 12 fields including `Vec<UserInput>` + multiple `Option<...>` configs)
- `codex-rs/tui/src/app_command.rs:191-228` — `AppCommandView<'a>` updated to mirror the new enum shape
- `codex-rs/tui/src/app_command.rs:297-485` — `into_core()` rewritten as exhaustive match producing `Op::*` values (the compatibility seam)
- `codex-rs/tui/src/app_command.rs:550-730` — `From<&AppCommand> for AppCommand` (clone path) and remaining helper impls updated
- 386 additions / 70 deletions, single file

## Verdict: **merge-after-nits**

## Rationale

Mechanically clean and correctly scoped to one file — exactly the right shape for a 7-step stack. The `#[allow(clippy::large_enum_variant)]` is the right tradeoff for now (boxing every variant would noise up call sites this stack is about to rewrite anyway), and the preserved `into_core()` keeps the blast radius zero outside this file. Concerns:

1. **`large_enum_variant` should not become permanent.** Add a `// TODO(stack-step-N): remove once UserTurn moves to thread-routing layer and this enum can shrink` next to the `#[allow]` at `app_command.rs:9` so the lint suppression has a known retirement plan and doesn't outlive the refactor.
2. **`UserTurn` and `OverrideTurnContext` are nearly identical.** `UserTurn` has 12 fields and `OverrideTurnContext` has 11 of the same fields wrapped in `Option`. Future steps may want a shared `TurnContextOverlay` struct so future-field additions only need to land in one place — note this in the stack tracking comment so it's not forgotten by step 7.
3. **`into_core()` is the only remaining `Op` coupling, and it's now ~190 lines of boilerplate match arms.** Worth a one-line code comment at `app_command.rs:297` saying "this entire impl block is removed in step N when X moves" so a reviewer of step 3+ can connect the dots without re-reading the stack description.
4. **Test coverage is implicit** (the PR body lists only `cargo check -p codex-tui`). Confirm the existing `chatwidget/tests/slash_commands.rs` and `tests/status_and_layout.rs` suites still run green — `cargo check` doesn't catch behavior regressions in the `into_core()` translation, and a typo there (e.g., flipping `approval_policy` and `permission_profile` in the `Op::OverrideTurnContext` arm at ~line 488) would silently change semantics for every subsequent turn.
5. Confirm Windows CI is green: the `OverrideTurnContext { windows_sandbox_level: ... }` field at `app_command.rs:64` only compiles on Windows builds; macOS/Linux smoke tests won't catch a typo there.
