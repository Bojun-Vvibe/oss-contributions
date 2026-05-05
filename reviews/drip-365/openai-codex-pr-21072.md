# openai/codex #21072 — [codex] Use backend collaboration modes in TUI picker

- **Head SHA:** `4a7184353c8526aaed40457c8ab35a3cccf52292`
- **Base:** `main`
- **Author:** canvrno-oai
- **Size:** +131 / −7 across 5 files (1 app, 1 session, 1 test helpers, 1 collaboration_modes, 1 model_catalog)
- **Verdict:** `merge-after-nits`

## Summary

Moves the source of truth for collaboration-mode discovery (the {Default, Plan,
…} mode picker) from a TUI-local `builtin_collaboration_mode_presets()` import
to the `collaborationMode/list` app-server RPC. TUI bootstrap now fetches the
mode list alongside the model list and stores it in `ModelCatalog`; the picker
reads from the catalog instead of the local builtins. Local builtins are kept
as a fallback only when the remote app-server is older and clearly lacks the
RPC.

## What's right

- **Tight detection of "old remote app-server".** The fallback gate at
  `app_server_session.rs:152-163` is a 3-clause match:
  `is_remote()` AND (`code == JSONRPC_METHOD_NOT_FOUND_ERROR_CODE` (-32601) OR
  (`code == JSONRPC_INVALID_REQUEST_ERROR_CODE` (-32600) AND message contains
  both `"collaborationMode/list"` AND `"unknown variant"`)). This correctly
  distinguishes (a) a JSON-RPC 2.0 method-not-found, (b) an enum-variant
  deserialization rejection that older `ClientRequest` enums emit when they
  don't know the variant, from (c) a real server error that happens to mention
  the method name. The third unit test at `app_server_session.rs:1614-1620`
  pins this third case explicitly: `"Experimental API … is not enabled"` is
  NOT a missing-method error and must propagate.

- **`is_remote()`-gated fallback is the right scope.** Embedded (in-process)
  app-servers are always built from the same source tree as the TUI, so a
  missing `collaborationMode/list` there is a real bug, not a version skew —
  worth surfacing loudly. Only remote (RPC-over-pipe) app-servers can be
  legitimately older. The `match` arm at line 156 enforces this.

- **`ModelCatalog::new(models, collaboration_modes)` is a clean ctor change.**
  Two-vec constructor at `model_catalog.rs:14-22`, single getter
  `list_collaboration_modes(&self) -> Vec<CollaborationModeMask>` at
  `model_catalog.rs:25-27`. The picker change at `collaboration_modes.rs:5-9`
  is a 3-line swap from `builtin_collaboration_mode_presets()` to
  `model_catalog.list_collaboration_modes()` with the existing
  `is_tui_visible` filter kept. No behavioral change for existing
  Default/Plan modes.

- **Bootstrap-shaped test fixtures stay backend-shaped.** Tests in
  `chatwidget/tests/helpers.rs:139-149` build a `Vec<CollaborationModeMask>`
  fixture from the existing `builtin_collaboration_mode_presets()` and strip
  `developer_instructions` (matching `collaboration_mode_mask_from_api_mask`
  at `app_server_session.rs:1115-1122` which sets `developer_instructions:
  None`). This keeps unit tests honest about what the API surface actually
  carries.

## Concerns / nits

1. **`developer_instructions: None` silently drops a field.** At
   `app_server_session.rs:1115-1122`, `collaboration_mode_mask_from_api_mask`
   maps the API `CollaborationModeMask` → protocol `CollaborationModeMask`
   and explicitly sets `developer_instructions: None`. If the backend ever
   ships a non-None `developer_instructions` (e.g. a server-side custom mode
   with bespoke instructions), the TUI silently drops them. This may be
   intentional today (the backend API mask doesn't carry that field), but
   the asymmetry should be a `// TODO: surface backend-provided
   developer_instructions when the API mask gains the field` comment so it
   doesn't get noticed only after a backend rollout adds it.

2. **Failure modes for non-`is_remote()` cases are conflated with hard
   bootstrap failures.** At `app_server_session.rs:166-171`, any error that
   isn't the missing-method case returns
   `bootstrap_request_error("collaborationMode/list failed during TUI
   bootstrap", err)` which (per `bootstrap_request_error` at line 130)
   wraps as a `color_eyre::Report`. This means a transient network blip
   during bootstrap fails the entire TUI startup rather than falling back
   to builtins (which is what the embedded path would have done before this
   PR). For embedded the strict-fail-on-error is correct (concern resolved
   by point 2 of "What's right"). For remote, it means TUI startup is now
   strictly more brittle: previously the picker worked from local presets
   regardless of app-server health, now a transient remote `collaborationMode/list`
   failure blocks TUI startup. Consider adding a transient-error escape
   that logs and falls back to builtins in the remote path.

3. **No `collaborationMode/list` request-id telemetry.** The request-id is
   minted via `self.next_request_id()` at line 240 but isn't surfaced in
   the error path or in the fallback debug log at line 158. If a user hits
   the rare third case (real server error not matching the missing-method
   pattern), debugging requires correlating the server's logs without
   knowing which client request id to grep for. One-line fix: include
   `request_id = collaboration_modes_request_id` in the
   `tracing::debug!`/`bootstrap_request_error` payloads.

4. **`CollaborationModeListParams::default()` is the only call site** at
   `app_server_session.rs:243`. If params ever grow non-defaultable fields,
   this becomes a silent API gap. Consider a one-line comment noting that
   the bootstrap intentionally requests the default (unfiltered) mode list.

5. **`builtin_collaboration_mode_presets()` import survives in two places**
   (`app_server_session.rs:107` and removed from `collaboration_modes.rs`).
   The second-line removal is correct (the picker no longer reads builtins
   directly). The first import remains because of the fallback path. Worth
   a comment in `collaboration_modes.rs:5-9` saying "do not re-add the
   builtins import here; the fallback is intentionally scoped to the
   bootstrap layer" so a future contributor doesn't restore symmetric usage.

## Verdict rationale

Architecturally correct (single source of truth at the API boundary),
backwards-compatible against older remote app-servers, well-tested for the
specific failure-detection logic. Concern 2 (transient-error brittleness on
the remote path) is the only one that affects user experience and is one
arm-addition away from being addressed.
