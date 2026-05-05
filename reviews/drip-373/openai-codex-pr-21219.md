# openai/codex #21219 — Add model and reasoning effort to turn metadata

- **Head:** `a46f9a3` (`a46f9a3aa29ccabeef0ee9ff18edd62ff7758b68`)
- **Repo:** openai/codex
- **Scope:** `codex-rs/core/src/turn_metadata.rs`, `turn_metadata_tests.rs`, `session/turn_context.rs`, `compact.rs`, `compact_remote_v2.rs`, `mcp_tool_call.rs`, `mcp_tool_call_tests.rs`, `session/turn.rs`, `session_startup_prewarm.rs`

## What the diff does

Adds two new keys (`model`, `reasoning_effort`) to the `X-Codex-Turn-Metadata` header that Codex emits with every turn-scoped outbound request (model API calls, MCP tool calls, sampling, compact, prewarm).

Load-bearing changes:

1. **`turn_metadata.rs:22-39`** — new `TurnMetadataRequest<'a> { model: &'a str, reasoning_effort: Option<ReasoningEffortConfig> }` struct and a private `TurnMetadataOverlay<'a>` that bundles all four overlay axes (model, reasoning_effort, turn_started_at_unix_ms, responsesapi_client_metadata).
2. **`turn_metadata.rs:96-130`** — `merge_turn_metadata` rewritten to take an overlay struct instead of two positional args. Crucially, the `responsesapi_client_metadata` filter at `:124-128` now skips reserved keys via a `matches!(key.as_str(), MODEL_KEY | REASONING_EFFORT_KEY | TURN_STARTED_AT_UNIX_MS_KEY)` instead of the previous single `key == TURN_STARTED_AT_UNIX_MS_KEY`. This is the right defensive choice — it prevents user-provided `responsesapi_client_metadata` from clobbering the canonical model/effort.
3. **`turn_metadata.rs:248-292`** — public API renamed: `current_header_value()` → `current_header_value_for_request(TurnMetadataRequest)`. Internal `build_current_header_value(Option<request>)` is the shared implementation. `current_meta_value()` is deleted; callers now compose `current_header_value_for_request(...)` + `serde_json::from_str` themselves (see `mcp_tool_call.rs:898-902`).
4. **`turn_context.rs:118-141`** — splits the existing `effective_reasoning_effort_for_tracing()` (which returned `String`) into two methods: `effective_reasoning_effort()` returning `Option<ReasoningEffortConfig>` (used by the new metadata path) and the existing tracing wrapper. New `turn_metadata_request()` constructor at `:135-140`.
5. All callers of the old `current_header_value()` / `current_meta_value()` are migrated (5 sites: `compact.rs:187`, `compact_remote_v2.rs:180`, `mcp_tool_call.rs:898`, `session/turn.rs:446`, `session_startup_prewarm.rs:225`).
6. **`mcp_tool_call_tests.rs:954-967`** — new asserts pin that the emitted metadata contains `model` matching `turn_context.model_info.slug` and `reasoning_effort` matching `turn_context.effective_reasoning_effort().to_string()`.

## Strengths

- The `TurnMetadataOverlay` struct refactor is pre-emptive: adding the next key (e.g. `personality`, `agent_id`) is now a one-line struct field + one-line merge insert, no more positional-arg explosion.
- The reserved-key filter widening at `merge_turn_metadata:124-128` is the security-relevant detail and is correctly done — without it, a misconfigured `responsesapi_client_metadata` could spoof the model field.
- `effective_reasoning_effort()` returns `None` (not `"default"`) for non-reasoning models. The header overlay only inserts the key when `Some`, so non-reasoning model turns won't carry a misleading `reasoning_effort: "default"` string.

## Concerns

1. **Breaking change to internal API.** `current_header_value()` and `current_meta_value()` are removed wholesale. Anything outside this PR's touched files that used them will break compilation. Confirm via `git grep` before merge that the migration is exhaustive.
2. **`mcp_tool_call.rs:898-902`** — manual `serde_json::from_str(&header).ok()` after `current_header_value_for_request` discards parse errors silently. The previous `current_meta_value()` did the same thing, so no regression, but this is a fragile pattern: if the merged header is ever malformed (e.g., a future overlay key inserts a non-JSON value), the request meta silently drops the entire turn metadata. A `tracing::warn!` on parse failure would surface the bug at the cost of one log line per turn.
3. **`mcp_tool_call_tests.rs:957-963`** — `effective_reasoning_effort().map(|effort| effort.to_string()).as_deref()` produces `Option<&str>` and asserts equality with `turn_metadata.get("reasoning_effort").and_then(Value::as_str)`. For the non-reasoning-model case both should be `None`, so the assert holds. Fine but an explicit `None`-side test case would harden it.
4. **Cardinality / privacy of `model` in metadata.** Exposing the exact model slug in every outbound MCP tool call header means any MCP server now learns which model the user is on. For most MCP servers this is benign (they need to size context for the caller anyway), but it's worth a one-line note in the PR body that this is intentional.

## Verdict

**merge-after-nits** — request a `tracing::warn!` on the silent `from_str(&header).ok()` drop at `mcp_tool_call.rs:902`, an explicit `None`-arm test for `reasoning_effort`, and a privacy-disclosure note in the PR body. Functionally the change is clean and the overlay refactor is a real maintainability win. Head `a46f9a3`.
