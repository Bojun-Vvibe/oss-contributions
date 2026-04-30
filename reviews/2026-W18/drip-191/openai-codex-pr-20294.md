# openai/codex #20294 — Add /ide context support to the TUI

- **URL:** https://github.com/openai/codex/pull/20294
- **Head SHA:** `3e37c2e6e9d1c49864e668f23f6f6c75c1980b10`
- **Files (diff slice):** `tui/src/bottom_pane/chat_composer.rs`, `tui/src/bottom_pane/footer.rs`, `tui/src/bottom_pane/mod.rs`, `tui/src/bottom_pane/slash_commands.rs`, `tui/src/chatwidget.rs`, `tui/src/chatwidget/ide_context.rs` (new), `tui/src/chatwidget/realtime.rs`, `tui/src/chatwidget/slash_dispatch.rs`, `tui/src/chatwidget/tests/composer_submission.rs`, `tui/src/chatwidget/tests/helpers.rs`, `tui/src/ide_context.rs` (new), `tui/src/ide_context/ipc.rs` (new, ~543 lines), `tui/src/ide_context/prompt.rs` (new) — total ~1797 diff lines
- **Verdict:** `needs-discussion`

## What changed

New `/ide` slash command that fetches active editor context (active file, selection, open tabs, optional `process.env.PATH`) over a per-uid Unix socket / Windows named pipe and prepends it to the next user message. Three new modules:

- **`tui/src/ide_context.rs`** — public data model (`IdeContext`, `ActiveFile`, `FileDescriptor`, `Range`, `Position`, `IdeProcessEnv`) with `serde(rename_all = "camelCase")` for the IPC wire shape, plus a deserialization regression test at `:777-810`.
- **`tui/src/ide_context/ipc.rs`** — request/response transport (~543 lines). Default socket path is `std::env::temp_dir().join("codex-ipc").join(format!("ipc-{uid}.sock"))` for unix at `:931-936` and `\\.\pipe\codex-ipc` for windows at `:939-941`. Frame size cap `MAX_IPC_FRAME_BYTES = 256 * 1024 * 1024` at `:836` (256 MiB). Request timeout `IDE_CONTEXT_REQUEST_TIMEOUT = 6s` at `:833`. `IdeContextError::is_retryable_after_recent_toggle` at `:866-883` enumerates exactly three retryable RequestFailed variants ("no-client-found", "client-disconnected", "request-timeout").
- **`tui/src/ide_context/prompt.rs`** — `apply_ide_context_to_user_input` at `:1377-1410` that prepends a `# Context from my IDE setup:\n...\n## My request for Codex:\n` block onto the first `UserInput::Text` item (or inserts a new one if none), with `text_elements` byte ranges shifted by `prefix_len` via `prefixed_text_input` at `:1427-1441`. Selection truncation cap `MAX_ACTIVE_SELECTION_CHARS = 200_000` at `:1374`. `extract_prompt_request_with_offset` at `:1416-1425` is the symmetric reverse used by `rendered_user_message_event_from_event` to *hide* the prepended block from the rendered history cell, asserted at `composer_submission.rs:1162-1185` and `:1186-1207`.

Footer surface: new `IdeContextStatusIndicator::Active` enum variant at `footer.rs:104-106`, rendered via the new `status_line_right_indicator_line` helper at `:573-603` that composites it with the existing collaboration-mode and goal-status indicators using ` · ` dim separators. `chat_composer.rs:387` adds the `ide_context_status_indicator: Option<IdeContextStatusIndicator>` field plus `set_ide_context_status_indicator` setter at `:723-728`.

Slash dispatch: `SlashCommand::Ide` arm in both `handle_command` and `handle_command_with_args` at `slash_dispatch.rs:603-605` and `:613-615`, plus addition to the always-allowed list at `:622-625`. Test helper at `helpers.rs:250` adds `ide_context: super::super::ide_context::IdeContextState::default()`.

## Why this is "needs-discussion" and not "merge-after-nits"

This is a 1.8k-line surface that introduces a new IPC transport, a privileged IDE-extension trust relationship, and silent prompt-injection of (potentially sensitive) editor content. Three things need a reviewer-with-context to weigh in before this lands:

1. **Trust boundary on the IPC socket.** The unix path `std::env::temp_dir().join("codex-ipc").join("ipc-{uid}.sock")` at `:931-936` lives in `/tmp`, which is world-traversable and (on multi-user macOS) shared between users. The `format!("ipc-{uid}.sock")` makes the *socket file name* uid-scoped but does not enforce that the socket *parent directory* `/tmp/codex-ipc` was created with mode 0700 and owned by the same uid. A second user on the same host can `mkdir /tmp/codex-ipc` first with their own perms, then `bind(2)` an `ipc-${their-uid}.sock` peer that the TUI doesn't talk to — but the diff slice doesn't show socket-credentials verification on the *client* side either, so a malicious peer that wins the create race for `/tmp/codex-ipc` controls subsequent socket placement. On Linux, `SO_PEERCRED` (or BSD `LOCAL_PEERCRED`) on the client side after `connect(2)` would confirm the server uid matches `getuid()`. The diff slice doesn't show this check; the windows named-pipe `\\.\pipe\codex-ipc` at `:940` similarly lacks a documented ACL check. This is the kind of thing where "no obvious bug visible in the slice" is not the same as "we know it's safe", because the threat model puts a privileged system-wide IPC channel in a shared namespace.

2. **256 MiB frame cap is two orders of magnitude too generous.** `MAX_IPC_FRAME_BYTES = 256 * 1024 * 1024` at `:836` is the hard limit on a single response frame from the IDE. The downstream `MAX_ACTIVE_SELECTION_CHARS = 200_000` at `prompt.rs:1374` truncates the *rendered prompt* to 200k chars, which suggests the design's actual selection-size budget is sub-1MiB. The 256 MiB cap appears to exist to tolerate `open_tabs` arrays from absurd workspace sizes, but it's also a memory-DoS amplifier: a misbehaving (or compromised) IDE extension can pin 256 MiB of TUI heap per `/ide` call. A `MAX_IPC_FRAME_BYTES` of 4–8 MiB with explicit `IdeContextError::ResponseTooLarge` (already wired at `:854`) telling the user "selection too large, narrow it" matches the prompt-side budget and removes the DoS amplification.

3. **No allow/deny redaction on `process_env.path`.** The `IdeProcessEnv { path: String }` field at `:766-769` is whatever the IDE sends; the diff slice doesn't show a redactor on it before it reaches the prompt-render path or the rollout log. On a corp-managed dev machine, `PATH` often encodes internal tooling paths, secret-store mounts, and user-identifying directory structure. If `render_prompt_context` (visible at `:1443-1514` but the env-rendering branch isn't shown in the slice; only `active_file`/`selection`/`open_tabs` rendering branches are) embeds `process_env.path` into the prompt, it becomes part of every model request the user sends after `/ide`. The PR description should explicitly state where `IdeProcessEnv` is consumed and whether it's user-toggleable.

## What is right

- The history-cell hide round-trip (`extract_prompt_request_with_offset` + the `text_elements` byte-shift at `prefixed_text_input` `:1427-1441`) is the right shape: the user sees only what they typed, the model sees the prepended context, and the rendered cell uses the symmetric reverse to extract the original message. The two regression tests at `composer_submission.rs:1162-1207` cover both the unit-level `rendered_user_message_event_from_event` path and the e2e `complete_user_message` → `UserHistoryCell::display_lines` path with snapshot `"› Ask Codex"`.
- The `is_retryable_after_recent_toggle` enumeration at `ipc.rs:866-883` is honest about exactly which failure modes are transient ("no-client-found" right after the user starts the IDE) versus permanent (`Connect`, `Send`, `InvalidResponse`, `ResponseTooLarge`). No retry-storm risk.
- `IdeContextError::user_facing_hint` at `:886-908` gives operators actionable guidance rather than a stack trace, with the `ResponseTooLarge` branch at `:899-901` already telling the user to clear the selection — which is the right UX for nit #2 above.
- `serde(rename_all = "camelCase")` plus `#[serde(default)]` on `open_tabs`, `process_env`, `start_line`, `end_line`, `selections`, `active_selection_content` at `:723-739` and `:746-751` makes the wire shape forward-compatible with IDE extensions that ship without these fields.
- The footer composition with ` · ` separators at `footer.rs:587-595` is a clean refactor: the old `status_line_right_indicator` at `chat_composer.rs:42-49` (deleted) was a 2-way `or_else`; the new `status_line_right_indicator_line` at `footer.rs:573-603` accepts an arbitrary indicator list and composes them. New snapshot at `bottom_pane/snapshots/codex_tui__bottom_pane__footer__tests__footer_status_line_enabled_mode_and_ide_context_right.snap` locks the rendering.

## Risk

High *footprint*, medium-high *threat surface*. The IPC layer is a brand-new privileged channel between the user's IDE and the TUI rollout-log path. The DoS-via-frame-size and IPC-trust-boundary concerns above are not "after-nit" because they affect every `/ide` invocation on every supported platform. If the maintainers can answer:

- (a) Does `connect(2)` on the unix socket verify peer uid via `SO_PEERCRED` / `LOCAL_PEERCRED`?
- (b) Does `\\.\pipe\codex-ipc` set a restrictive DACL via `SECURITY_ATTRIBUTES` at create time, and does the client verify the server SID on connect?
- (c) Does `render_prompt_context` consume `process_env.path`, and if so is there a config gate or redactor?
- (d) Is the 256 MiB cap intentional, and what's the largest legitimate response observed in pre-merge testing?

…then this can move to `merge-after-nits` (frame-cap reduction + maybe a `--ide-context-strict` flag for the env path). Without those answers, this is a substantial new attack surface to merge sight-unseen.
