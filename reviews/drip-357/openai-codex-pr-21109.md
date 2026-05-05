# openai/codex #21109 — feat(tui): add local file upload command

- **Repo**: openai/codex
- **PR**: [#21109](https://github.com/openai/codex/pull/21109)
- **Author**: efrazer-oai
- **Head SHA**: `09f54d7f020da76e19fc80b2e608fb0f745043e4`
- **Base branch**: `efrazer/codex/remote-file-uploads-app-server` (stacked PR)
- **Created**: 2026-05-04T23:53:28Z

## Verdict

`merge-after-nits`

## Summary of change

Adds a `/upload <local-path>` slash command to the Codex TUI that reads a
client-local file, base64-encodes it, ships it to the host via the existing
`fs/uploadFile` JSON-RPC method, and inserts the returned host path into the
composer. The PR is the TUI half of a stack — the app-server side
(`FsUploadFile*` types, host-managed upload storage) was added in the parent
PR.

Touched surface:

- `tui/src/slash_command.rs:60` — new `Upload` variant in the
  `SlashCommand` enum, with `:102` description "upload a local file to the
  app-server host", and `:161` and `:210` adding it to two existing
  variant-list helpers (presumably "commands needing args" and
  "commands available in the dispatcher").
- `tui/src/chatwidget/slash_dispatch.rs:34` — new `UPLOAD_USAGE` constant
  `"Usage: /upload <local-path>"`.
- `tui/src/chatwidget/slash_dispatch.rs:377-379` — bare `/upload` (no arg)
  prints the usage hint via `add_info_message`.
- `tui/src/chatwidget/slash_dispatch.rs:739-743` — `/upload <trimmed>` with a
  non-empty arg fires `AppEvent::UploadLocalFile { path: PathBuf::from(trimmed) }`.
- `tui/src/app_event.rs:215-223` — two new `AppEvent` variants:
  `UploadLocalFile { path }` and `LocalFileUploaded { local_path, result: Result<PathBuf, String> }`.
- `tui/src/app/event_dispatch.rs:651-665` — handlers for both events.
  Success → `chat_widget.insert_uploaded_file_path(&remote_path)`; failure →
  `add_error_message(format!("Failed to upload '{}': {err}", local_path.display()))`.
- `tui/src/app/background_requests.rs:102-112` — `App::upload_local_file`
  spawns a tokio task; `:559-583` is the `upload_local_file_request` async
  function: it reads `local_path` via `tokio::fs::read`, base64-encodes with
  `STANDARD`, builds an `FsUploadFileParams { file_name, data_base64 }`, and
  calls `request_handle.request_typed(ClientRequest::FsUploadFile { ... })`.
- `tui/src/chatwidget.rs:10382-10387` — `insert_uploaded_file_path` inserts
  a leading space if the composer text is non-empty, then inserts the
  returned path.
- `tui/src/chatwidget/tests/slash_commands.rs:1929-1955` —
  `queued_upload_stops_before_follow_up_prompt`: when a queued `/upload
  /tmp/demo.txt` is followed by a queued user message "use the upload", the
  test asserts an `AppEvent::UploadLocalFile { path: PathBuf::from("/tmp/demo.txt") }`
  is emitted, the follow-up message stays in the queue (`queued_user_messages
  .len() == 1`), and no model op is sent yet. Good queue-interaction
  coverage.

## What's good

- Cleanly fits the existing `AppEvent` pattern: synchronous "kick off" event
  + async "result" event — no shared mutable state, no callback handling
  in the spawned task beyond a typed response.
- Stays aligned with the TUI's existing path-reference model: after upload,
  the user sees a normal path that the host can read later. No new mental
  model.
- Per-request id `format!("upload-local-file-{}", Uuid::new_v4())` (`:572`)
  is descriptive enough to grep for in app-server logs.
- Error path is end-user-friendly: `"Failed to upload '{local_path}': {err}"`
  surfaces in the chat as an error message, not a panic.
- The queued-command test on `:1929-1955` is exactly the right shape — it
  proves the upload starts as a side effect *without* draining the queued
  user prompt prematurely, which is the trickiest part of slash-command
  ordering.

## Nits / concerns

- **No size cap.** `tokio::fs::read(local_path).await?` reads the entire
  file into memory and then base64 encodes it (`+33%` overhead) before
  shipping it as a single JSON-RPC request. A user typoing
  `/upload /var/log/system.log` against a multi-GB log will OOM the TUI.
  Add an upper bound (e.g. 100 MB) with a clear error message before the
  read, or stream via the (future) chunked-upload protocol.
- **No path normalization or tilde-expansion.** `PathBuf::from(trimmed)`
  on `:741` accepts whatever the user typed; `~/file.txt` will be passed
  through as a literal `~`-prefixed name and `tokio::fs::read` will fail
  with ENOENT. Likewise relative paths are interpreted relative to the TUI's
  cwd, which may not match the user's mental model. Consider routing through
  `shellexpand::tilde` and resolving against the session cwd, with the
  resolved path echoed in any error message.
- **`file_name` extraction silently fails on bare paths.**
  `local_path.file_name().and_then(|name| name.to_str())` (`:565-568`)
  returns `None` for paths ending in `/` or with non-UTF-8 components,
  yielding the error `"Upload path must name a file."` — a fine error, but
  the message would benefit from echoing the offending path.
- **Insertion behavior on a non-empty composer**: `insert_uploaded_file_path`
  prepends a single space (`:10384`). If the cursor is mid-word, this will
  produce `foo /host/path/file.txt bar`, breaking the word the user was
  typing. Consider also appending a trailing space, or inserting at a
  word-boundary-aware position.
- **`base64::engine::general_purpose::STANDARD`** uses padding; many
  JSON-RPC peers prefer `STANDARD_NO_PAD` or URL-safe variants. Confirm
  the app-server side decodes with the matching engine; the parent PR
  presumably specifies this, but it's not visible from this diff alone.
- **Concurrent-upload behavior is unspecified.** Two `/upload` invocations
  fired in quick succession will spawn two tasks racing for the composer
  insertion point. Probably fine for a TUI workflow, but worth a comment.

## Sanitization

No secrets or tokens in the diff. Test paths use `/tmp/demo.txt`. Safe.

## Risk

Medium. The base flow is correct and well-tested for the queue-interaction
case, but the lack of a file-size cap is a real foot-gun for end users on
large files.

## Recommendation

Land after (a) a defensive size cap with a clear error, and (b) at least
tilde-expansion for the path argument so `/upload ~/my.txt` works the way a
user expects. The other nits are polish.
