---
pr-url: https://github.com/openai/codex/pull/20278
sha: 87d0cf1a62e9
verdict: merge-after-nits
---

# feat: Add workspace plugin sharing APIs

Large feature surface: 2402+/108− across 80+ files adding three v2 RPCs — `plugin/share/save`, `plugin/share/list`, `plugin/share/delete` — plus the supporting protocol schemas, message-processor wiring, and remote-share HTTP client. The load-bearing implementation is `codex-rs/core-plugins/src/remote/share.rs:1-418`: `save_remote_plugin_share` archives a local plugin root with `flate2::write::GzEncoder` on a `tokio::task::spawn_blocking`, enforces `REMOTE_PLUGIN_SHARE_MAX_ARCHIVE_BYTES = 50 * 1024 * 1024` (declared at `:18`), then walks a three-step upload flow — `POST /backend-api/public/plugins/workspace/upload-url` to get a presigned URL + ETag, `PUT` the archive bytes there, then `POST` create/update with `(file_id, etag)`. The optional `remote_plugin_id` toggle lets the same RPC update an existing share rather than creating a new one (covered by the `plugin_share_save_uploads_local_plugin` integration test using `wiremock::MockServer`).

Schema discipline is exemplary — every new RPC gets its `Params`/`Response` types in three formats (`json`, `typescript`, plus the v2 separation) with the v2 carve-out registered at `app-server-protocol/src/protocol/v2.rs`. The deletion at the v2 `plugin_install` arm of the prior `plugin_name.starts_with("plugins~") || ... starts_with("connector_")` allowlist is a corollary of the new flow: the share/save round-trip now produces canonical IDs through the create endpoint so the client doesn't need to whitelist prefixes anymore — the validation moves to the server. That's the right direction (server validates what server owns) but worth calling out because the previous error message at the deleted site was load-bearing diagnostic UX (`"invalid plugin id: expected ... starting with plugins~, app_, asdk_app_, or connector_"`); the new path needs the equivalent shape on the failure case.

Three nits:

1. **50MB archive limit is enforced post-archive, not pre-archive** — the `archive_plugin_for_upload` call at `:215` reads the entire plugin root into memory and gzips it before checking size. A 200MB plugin root with a `node_modules`-shaped tree will OOM before the limit fires. A pre-walk size estimate (sum of `metadata().len()` across the include set) gated against `REMOTE_PLUGIN_SHARE_MAX_ARCHIVE_BYTES * 2` (2x margin for compression ratio) would fail-fast.
2. **No `Cache-Control`/`If-Match` on update** — when `remote_plugin_id` is set, the create-with-etag flow doesn't pass an `If-Match` header for the *plugin* (only the upload object), so two concurrent saves to the same `remote_plugin_id` race with last-write-wins semantics. For a "share my plugin" UX that's probably fine but should be documented.
3. **`spawn_blocking` for archive but `await` for upload** — the `tokio::task::spawn_blocking` correctly avoids parking the runtime during gzip, but then the upload itself is a single sequential `PUT`. For 50MB archives over a slow link that's a 30s+ blocking RPC from the IDE's perspective with no progress notification streamed back. Worth a follow-up that emits `plugin/share/saveProgress` events.

## what I learned
For "upload-then-create" flows the size limit *must* be enforced before reading the source bytes, not after compressing them. The compression step looks like it bounds memory but actually doubles it (source bytes + compressed bytes both held), and for any "include the whole directory" archive shape there's no upper bound on the source set without explicit gating. The two-phase commit (presigned URL, then create with `(file_id, etag)`) is the right server contract — it lets the server validate the archive existence before committing the metadata row, so a torn upload doesn't leave a half-created plugin record.
