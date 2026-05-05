# openai/codex PR #21108 — feat(app-server): add managed remote file uploads

- Repo: `openai/codex`
- PR: #21108
- Head SHA: `43b3c03dc2e0`
- Author: `efrazer-oai`
- Scope: 16 files, +445 / -5
- Verdict: **merge-after-nits**

## What it does

Adds a new `fs/uploadFile` v2 protocol method (paired schema files for both JSON and TS, plus regenerated combined schemas) that lets clients ship a base64-encoded file payload to the app-server, which then writes it into a per-upload UUIDv7-named subdirectory under `${codex_home}/uploads/<uuid>/<sanitized-filename>` and returns the resolved absolute path.

Wiring:

- Schemas: `app-server-protocol/schema/json/v2/FsUploadFileParams.json`, `FsUploadFileResponse.json`, plus the regenerated combined `ClientRequest.json`, `codex_app_server_protocol.schemas.json`, and `codex_app_server_protocol.v2.schemas.json` get a new `fs/uploadFile` discriminant.
- Rust types: `app-server-protocol/src/protocol/v2.rs` and `common.rs` get `FsUploadFileParams { fileName, dataBase64 }` / `FsUploadFileResponse { path }`.
- Processor wiring: `app-server/src/message_processor.rs` (~lines 447, 892) routes `fs/uploadFile` requests, and `request_processors/fs_processor.rs` carries the implementation:
  - `FsRequestProcessor::new(... codex_home: PathBuf)` — gets the codex home injected
  - `FsRequestProcessor::upload_file(...)` (line ~95) — sanitizes filename → decodes base64 with size cap → mkdir-p the per-upload dir → write file → return absolute path
  - `sanitize_upload_file_name` (line ~232) — `Path::file_name()` strip + non-empty check, defends against `../etc/passwd`-style payloads by keeping only the basename
  - `decode_upload_data_base64_with_limit` (line ~243) — pre-flight checks the **base64 length** against `max_base64_len = max_bytes.div_ceil(3) * 4` so an oversized request is rejected before allocating the decoded buffer; double-checks the decoded byte length after
  - `MAX_UPLOAD_FILE_BYTES: usize = 50 * 1024 * 1024` (line ~529 in the diff context) — 50 MB hard cap
- Tests: in-module test `upload_decode_rejects_payloads_larger_than_limit` plus an integration test in `app-server/tests/suite/v2/fs.rs:301-336` that round-trips a real upload and asserts the file lands under `codex_home/uploads/`. Test helper `send_fs_upload_file_request` added to `tests/common/mcp_process.rs:989-1002`.
- README updated for the new method.

## What's good

- **Path-traversal hardening is in the right place.** `sanitize_upload_file_name` keeps only the final path component (`Path::new(file_name).file_name()`), so `fileName: "../../etc/passwd"` collapses to `"passwd"` and lands inside the per-upload UUID dir, not in `$HOME/etc/`.
- **Per-upload UUIDv7 directory** means two clients uploading `report.pdf` simultaneously don't collide and don't leak each other's filenames into a shared path.
- **Pre-decode size check** (`max_base64_len` math at `fs_processor.rs:~248`) prevents a 1 GB base64 string from being decoded into RAM just to be rejected — this is the right way to bound `STANDARD.decode` cost.
- **Schema regeneration is in the same PR**, so codegen consumers stay coherent with the Rust definition.

## Nits / discussion points

1. **No retention / cleanup story.** Files written under `${codex_home}/uploads/<uuid>/` are persistent for the lifetime of the install. There is no TTL, no per-thread reaper, no API to delete. A long-lived install that uploads frequently will accumulate disk usage forever. Recommend at minimum: (a) a follow-up `fs/deleteUpload` method, or (b) tying the upload dir to a thread/turn so it can be GC'd with the thread.

2. **Hardcoded 50 MB cap, no config knob.** `MAX_UPLOAD_FILE_BYTES = 50 * 1024 * 1024` is a magic constant in `fs_processor.rs`. For headless / CI / vision workflows this might be too small or too large; for a public-facing app-server it's also the only DoS bound. Suggest making this a `config.toml` key (`app_server.max_upload_bytes`) or at least a `pub const` documented in the README. Easy follow-up.

3. **Filename sanitization is pure-basename.** That's correct against path traversal but it does mean two requests with the same `fileName` land at different absolute paths (different UUID dirs). Worth one line in the README clarifying that the response `path` is the canonical handle clients should retain.

4. **Base64 length math edge case.** `max_base64_len = max_bytes.div_ceil(3) * 4` is the **unpadded-rounded-up-to-quartet** length, which is correct for standard base64; a payload using URL-safe base64 or one with whitespace would fail decode for a separate reason, but a payload that is e.g. `max_bytes=50MB` and includes line wrapping (`\n` every 76 chars per RFC 2045) would exceed `max_base64_len` purely from the wrappers and get rejected before decode. In practice MCP/JSON-RPC clients don't wrap, but worth a one-line comment that wrapped base64 is rejected at the length check rather than at decode.

5. **No content-type / extension policy.** The server happily accepts and stores any bytes under any filename. That's defensible (caller knows what they're shipping), but worth surfacing in the README so callers know the server doesn't sniff or validate the payload type.

## Verdict
**merge-after-nits** — the path-traversal handling and pre-decode size check are correct, the schema regen is honest, and the integration test exercises the happy path. The retention/cleanup gap (#1) and the hardcoded cap (#2) are the things to follow up on, but neither blocks this PR shipping the protocol method.
