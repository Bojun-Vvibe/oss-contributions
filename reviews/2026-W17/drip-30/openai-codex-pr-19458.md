# openai/codex#19458 — ChatGPT Library file upload/download hooks

**Verdict: needs-discussion**

- PR: https://github.com/openai/codex/pull/19458
- Author: lt-oai
- Base SHA: `8a559e7938bd841d78dc2c442deaa76424faf1d9`
- Head SHA: `75102a2dabf929d305fba35e948fe016460f8302`
- +912 / -29

## Summary

Adds a "store_in_library" upload mode to `codex-api/src/files.rs` plus a
new `codex-rs/core/src/codex_apps_file_download.rs` (+324) module that
downloads files referenced by `sediment://`-style URIs from the host's
ChatGPT Library backend through the MCP tool-call layer. `UploadedOpenAiFile`
gains a `library_file_id: Option<String>` and changes `download_url:
String` → `download_url: Option<String>`. Upload flow forks: the legacy
`/files/{id}/uploaded` finalize path stays for non-library uploads; a
new `/files/process_upload_stream` SSE-style endpoint is consumed for
library uploads, parsing newline-delimited JSON status frames for
`metadata_object_id` / `library_file_id`.

## Specific findings

- `codex-rs/codex-api/src/files.rs:+487/-18` (head SHA
  `75102a2dabf929d305fba35e948fe016460f8302`) — `UploadedOpenAiFile`
  field change `download_url: String → Option<String>` is a
  source-breaking change for every internal consumer. The new
  `OpenAiFileUploadOptions { store_in_library: bool }` is `Default`-able
  and threaded through `upload_local_file(.., options: &OpenAiFileUploadOptions)`,
  which is also a signature break — every caller now needs an extra
  argument. Worth confirming in the PR description that the call-sites
  audit is complete (the diff touches `mcp_openai_file.rs` and
  `mcp_tool_call.rs`, but I'd want to see no other callers of
  `upload_local_file` outside this PR).
- `codex-rs/codex-api/src/files.rs:~+340` (the new `process_upload_stream`
  helper) — parses the response as line-delimited JSON status frames:
  ```
  for line in process_body.lines() {
      let line = line.trim();
      ...
  }
  ```
  This consumes the *entire* body before parsing, defeating the
  streaming contract of `process_upload_stream`. For very large library
  uploads this will block on the full body and miss intermediate
  progress events. Consider switching to a true streaming reader
  (`response.bytes_stream()` + line-buffer) — the
  `futures::StreamExt` import is already added but only used for the
  `download_to_path` chunked write below.
- `codex-rs/core/src/codex_apps_file_download.rs:1` (+324, new file) —
  introduces a 324-line module just for the download-to-path codex_apps
  flow without unit tests. The change set adds 16 lines to
  `mcp_tool_call_tests.rs`, but the new module itself has no test
  coverage in this PR. For a 324-line networking module that handles
  `sediment://` URI parsing, redirect handling, and on-disk writes with
  size limits, this is a coverage gap.
- `codex-rs/core/src/mcp_tool_call.rs:+44/-3` — wires the download hook
  into the MCP tool-call result post-processing path. The +16 test
  delta in `mcp_tool_call_tests.rs` covers the happy path, but I don't
  see negative coverage for the `OpenAiFileError::WriteFile` /
  `InvalidUrl` arms newly added to the error enum.
- `codex-rs/utils/plugins/src/mcp_connector.rs:+5` — five-line plugin
  surface change to expose the new file-handling capability. Worth
  cross-checking that this doesn't re-export any sediment URL handling
  to plugins that aren't supposed to see it.

## Rationale

This is a genuinely large feature land (+912 lines, new module, new
backend protocol, SDK error enum growth, breaking signature change on
`upload_local_file`) and it should not slip in without explicit
discussion. Three substantive concerns, in priority order: (1) the
process_upload_stream consumer reads the whole body before parsing,
which negates streaming semantics — operators with multi-hundred-MB
library uploads will see a long stall with no progress reporting. (2)
324 lines of new download-path code with zero direct unit tests; the
existing `mcp_tool_call_tests.rs:+16` covers integration but not the
URI parsing, size-limit enforcement, write-path error mapping, or the
new `library_file_id`-bearing `UploadedOpenAiFile` shape. (3) The
breaking signature change on `upload_local_file` — adding a required
`&OpenAiFileUploadOptions` argument rather than a new
`upload_local_file_with_options` constructor or builder — forces every
downstream consumer (including any external embedders of `codex-api`)
to chase the new arg. A `Default`-able options struct mitigates
ergonomics but doesn't fix the source break. The library-upload
direction itself is clearly correct and the PR description names the
right backend endpoints, but the magnitude of the change deserves a
maintainer thread before merge — especially around the streaming
parser and the test gap. Verdict reflects that, not a structural
objection to the feature.
