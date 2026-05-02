# openai/codex PR #20709 — Use responses request helpers for compact requests

- URL: https://github.com/openai/codex/pull/20709
- Head SHA: `094f269164afb470a235820db9698356acbfd4e0`
- Author: aibrahim-oai
- Verdict: **merge-after-nits**

## Summary

Refactors the compaction endpoint client (`CompactClient`) to share request-building helpers with the responses client instead of carrying its own bespoke `CompactionInput` shape and ad-hoc header assembly. Two new pub(crate) helpers are extracted in `requests/responses.rs`: `build_responses_request_body` (handles JSON serialization plus the Azure-specific `attach_item_ids` step) and `build_responses_request_headers` (handles `x-client-request-id`, conversation headers, and the optional `x-openai-subagent` header). `CompactClient::compact_input` is replaced by `compact_request`, which now takes a full `ResponsesApiRequest` + `ResponsesOptions`. The `CompactionInput` struct and its public re-export are deleted.

## Line-level observations

- `codex-rs/codex-api/src/requests/responses.rs` lines 19–43: the two new helpers are clean and well-scoped. `build_responses_request_body` correctly preserves the Azure branch (`provider.is_azure_responses_endpoint()` → `attach_item_ids`). `build_responses_request_headers` consumes `extra_headers` by `mut` parameter rather than re-allocating — efficient and matches the pattern in the responses client.
- `codex-rs/codex-api/src/endpoint/responses.rs` lines 79–84: the inline body/header construction is replaced with calls to the two helpers. The behavior should be byte-identical because the helper bodies are literal cut-paste of the original. Diff confirms `attach_item_ids` is removed from `requests/mod.rs`'s pub(crate) re-export — that removal is only safe if the only call site outside `responses.rs` was `responses.rs` itself. Verified from the diff: yes, the re-export is gone and there are no other callers.
- `codex-rs/codex-api/src/endpoint/compact.rs` lines 50–67: `compact_request` is the new entrypoint. It destructures `ResponsesOptions { conversation_id, session_source, extra_headers, .. }` — the `..` discards `compression` and `turn_state`. That is correct because compaction does not stream/compress, but it means a future `ResponsesOptions` field will be silently ignored on the compact path. Worth a `// NOTE: compaction ignores compression + turn_state` comment so this isn't lost.
- `codex-rs/codex-api/src/endpoint/responses.rs` line 28: `ResponsesOptions` gains `#[derive(Clone)]`. This is needed because callers can now hand the same options struct to both `responses_client.stream` and `compact_client.compact_request`. Reasonable, but reviewers should check `extra_headers` (`HeaderMap`) and any inner `Arc` fields: cloning a large `HeaderMap` per request is non-trivial. Probably fine for the typical ~5-10 header case but worth a quick measurement if this is on a hot path.
- `codex-rs/codex-api/src/common.rs` lines 23–37: `CompactionInput<'a>` is deleted. Same struct removed from `lib.rs` re-exports. **This is a breaking change to the `codex-api` crate's public API.** Anyone outside this repo depending on `codex_api::CompactionInput` will break at compile time. If `codex-api` has external consumers (or is published), this needs a changelog/version bump callout. If it's internal-only, no action needed.
- The PR removes the standalone `compact_input` method (`compact_input(&CompactionInput, HeaderMap)`) entirely — there is no deprecated shim. Reasonable for an internal refactor; risky if the surface is consumed externally.

## Suggestions

1. Add a brief comment on `compact_request` documenting which `ResponsesOptions` fields it consumes vs ignores (compression, turn_state). Future drift here would be silent.
2. If `codex-api` is treated as a public crate, this PR should carry a changelog entry noting `CompactionInput` removal.
3. Consider whether `build_responses_request_body` should also take an `Option<&dyn Fn(...)>` hook for provider-specific transforms beyond just Azure's `attach_item_ids` — not needed today, but the helper is now the obvious extension point.
4. Optional unit test: feed a `ResponsesApiRequest { store: true, ... }` through both helpers with an Azure provider and assert the resulting body has item IDs attached. Covers the only conditional branch in `build_responses_request_body`.
