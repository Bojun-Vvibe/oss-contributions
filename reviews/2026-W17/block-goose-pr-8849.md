# block/goose #8849 — feat(acp): migrate system/file Tauri commands to ACP+

- **Repo**: block/goose
- **PR**: #8849
- **Author**: fresh3nough
- **Head SHA**: c5e13ee73dc8a14bfe74183bd92be39f3ddc6e49
- **Base**: main
- **Size**: +1133 / −221 across the SDK request types
  (`crates/goose-sdk/src/custom_requests.rs`), the goose crate
  (new `mime_guess = "2"` dep + handler impls), the ACP meta/schema
  JSON, and the `goose2` desktop UI shell.

## What it changes

Adds 7 new ACP RPC methods under the `_goose/system/*` namespace,
defined as paired `JsonRpcRequest`/`JsonRpcResponse` types in
`custom_requests.rs:465-602`:

- `_goose/system/home_dir` → `GetHomeDirResponse { path }`
- `_goose/system/path_exists` → `PathExistsResponse { exists }`
- `_goose/system/list_directory_entries` → `Vec<FileTreeEntryDto>`
  with `kind: "file" | "directory"`
- `_goose/system/inspect_attachment_paths` → batched
  `AttachmentPathInfoDto[]` with optional `mime_type`, missing
  paths silently skipped
- `_goose/system/list_files_for_mentions` → walker for `@`-mention
  pickers, honours `.gitignore`, hidden files, symlink escapes;
  `max_results` clamped to 1..=5000 (default 1500)
- `_goose/system/read_image_attachment` → base64 + `mime/image/*`
- `_goose/system/write_file` → bytes-to-disk with parent-dir creation,
  scoped to "writes the desktop shell delegates after a native dialog"

The desktop shell changes ride on the ACP+ contract via the
auto-generated SDK types; the previous `tauri::command` handlers
in `goose2/` are removed in favor of these RPC calls.

## Strengths

- Clean type discipline: every request/response uses
  `#[derive(JsonRpcRequest)]` / `#[derive(JsonRpcResponse)]` plus
  `JsonSchema` so the wire schema and TS bindings are generated, not
  hand-written. The schema additions in
  `crates/goose/acp-schema.json:1390-…` line up 1:1 with the Rust types.
- `kind: "file" | "directory"` as a string discriminator at
  `custom_requests.rs:494` is fine for a stable, tiny enum and avoids
  the JSON-Schema gymnastics of a tagged enum across an SDK boundary —
  but it does push validation to the consumer (see ask below).
- `inspect_attachment_paths` choosing to *silently skip* missing
  entries (per the docstring at `:514`) is the right UX call for an
  attachments preview pane: a stale clipboard path shouldn't fail
  the whole batch. The contract is documented in the type, not just
  in the handler implementation.
- The `max_results` clamp in `list_files_for_mentions`
  (`:551-555`) bounds memory / stream length — important because
  this is hit on every keystroke after `@`. The default of 1500 and
  ceiling of 5000 are sensible.
- `mime_guess = "2"` (`crates/goose/Cargo.toml:196`) is the standard
  Rust crate for extension-based MIME detection — no content sniffing,
  no panics on unknown extensions, deterministic.

## Concerns / asks

- `kind: String` everywhere it appears (`FileTreeEntryDto.kind`,
  `AttachmentPathInfoDto.kind`) is an unconstrained string in the
  generated JSON schema. Consider `enum`-typing the schema with
  `["file", "directory"]` — same Rust signature, but consumers get
  better TS unions and the schema documents the contract. A typo in
  any future server-side handler (`"dir"` instead of `"directory"`)
  would currently slip through.
- `_goose/system/read_image_attachment` (`:583-602`) returns the
  whole base64 blob in a single response. For a 10MB image that's
  ~13MB on the wire and a single allocation in both the handler and
  the SDK consumer. Worth a comment that this RPC is for
  "user-attached image previews, not arbitrary file reads" plus a
  byte-size cap (e.g. 25MB) with a structured error when exceeded —
  otherwise it's a foot-gun if a future feature passes a large path
  through it.
- `_goose/system/write_file` (`:610-619`) takes `contents: String`,
  i.e. UTF-8 only. The docstring scopes it to "exported sessions",
  which is fine for that use case, but the type name doesn't hint at
  the UTF-8 restriction — a future caller trying to write binary will
  get a confusing serialization error far from the call site. Consider
  renaming to `WriteTextFileRequest` or adding a `Vec<u8>` variant
  for binary.
- `list_files_for_mentions` honoring `.gitignore` is great for the
  default repo-aware case, but the request type doesn't expose any
  toggles (e.g. include hidden, follow symlinks, additional ignore
  patterns). The walker behavior is hardcoded in the handler — fine
  for v1, but the request type should probably reserve the room
  with `#[serde(default)] pub include_hidden: Option<bool>` etc., so
  adding them later isn't a wire-breaking change.
- The request type names mix conventions: `GetHomeDirRequest` vs
  `PathExistsRequest` vs `ReadImageAttachmentRequest` — verb
  placement is inconsistent. Pick a convention (e.g. always
  `<Verb><Object>Request`) and rename now while these types are
  brand-new.

## Verdict

**merge-after-nits** — the contract design is clean, the namespace
grouping (`_goose/system/*`) is correct, and the SDK-as-source-of-
truth pattern is well executed. Asks are all about hardening for
the long tail: enum-typed `kind`, byte caps on the image read,
explicit text vs binary on the write path, reserved options on
the mention walker, and a one-pass naming sweep.

## What I learned

The ACP+ migration trades Tauri's tight coupling between the desktop
shell and the goose binary for a stable JSON-RPC contract that any
ACP-speaking client can implement. The cost is that what used to
be a `tauri::command` returning a typed Rust value now needs full
JSON Schema discipline — which catches more bugs at the boundary
but also exposes string-discriminator and unbounded-payload
ergonomic issues that a same-process call would have hidden.
