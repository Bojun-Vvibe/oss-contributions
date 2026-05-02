# openai/codex PR #20719 — Use responses request helpers for compact requests

- **URL**: https://github.com/anomalyco/codex/pull/20719
- **Head SHA**: `ee5d10d338cf`
- **Files touched**: `codex-rs/codex-api/src/common.rs`, `codex-rs/codex-api/src/endpoint/compact.rs`, `codex-rs/codex-api/src/endpoint/responses.rs`, `codex-rs/codex-api/src/lib.rs`, plus new `requests/responses.rs` helper

## Summary

Removes the bespoke `CompactionInput` struct and the inline body/header construction inside `responses.rs`. Both `/responses` (streaming sample) and `/responses/compact` now go through `build_responses_request_body` + `build_responses_request_headers`. To make the shared body work, several `ResponsesApiRequest` fields (`store`, `stream`, `include`) become `Option<...>` with `skip_serializing_if` so compact can blank them out without sending defaults.

## Comments

- `common.rs:46-54` — the doc comment "compact starts from this same shape and then clears the fields that endpoint does not accept" is the only place the contract is stated. With `store`/`stream`/`include` now `Option`, *every* future field author has to remember to make compact-incompatible additions optional too. Encode this as a separate `CompactSafeRequest` newtype, or at minimum a unit test that round-trips the compact body and asserts the forbidden fields are absent.
- `common.rs:54-65` — flipping `store: bool` → `store: Option<bool>` is a wire-format change for any consumer that was deserializing this struct. If `ResponsesApiRequest` is `pub`, this is a semver break. Confirm it's `pub(crate)` or document the bump.
- `common.rs:62` — `tool_choice: String` is removed entirely from `ResponsesApiRequest` and `ResponseCreateWsRequest`. Was that intentional, or carried in via another path? The diff doesn't show where `tool_choice` is now plumbed through; if it's silently dropped on the wire, this regresses tool-forcing behaviour.
- `endpoint/compact.rs:113-133` — `compact_request` now takes `ResponsesOptions` but only consumes 3 of its 5 fields (`conversation_id`, `session_source`, `extra_headers`). The destructure uses `..` to discard `compression` and `turn_state`. If a future caller assumes "compact honours my ResponsesOptions", they'll be surprised. Either accept a narrower `CompactOptions` struct or add a `debug_assert!(options.turn_state.is_none())`.
- `endpoint/responses.rs:181-183` — Azure-specific `attach_item_ids` was previously gated by `request.store && is_azure_responses_endpoint()`; the new `build_responses_request_body` call site has to preserve that exact guard. Worth grepping for `attach_item_ids` to confirm it moved into the helper rather than getting dropped.
- The `ResponsesOptions` `#[derive(Clone, Default)]` change (line 159) is needed because compact and responses both consume it now — fine, but `Clone` on a struct that owns `extra_headers: HeaderMap` means a per-call allocation. Cheap, but worth noting.

## Verdict

`request-changes` — three concrete blockers: (1) confirm `tool_choice` removal is intentional and tested, (2) confirm `attach_item_ids` Azure path is preserved inside the new helper, (3) document/encode the "compact must keep new fields optional" invariant beyond a doc comment. Refactor is otherwise clean.
