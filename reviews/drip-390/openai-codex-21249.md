# Review: openai/codex#21249 — Propagate cache key and service tiers in compact

- Head SHA: `f33af039b45c70474076607f2be1879f2a16fbd0`
- Files: 8 (+563 / -11)
- Verdict: **merge-after-nits**

## Summary

Adds `service_tier` and `prompt_cache_key` to the `/responses/compact` endpoint payload so compaction preserves the request-affinity fields that normal sampling already has. ChatGPT-auth gets both; API-key auth gets only `prompt_cache_key`. Follows the request-parity direction from #20719.

## Specific evidence

- **`CompactionInput` schema extension** at `codex-rs/codex-api/src/common.rs:35-38`:
  ```rust
  #[serde(skip_serializing_if = "Option::is_none")]
  pub service_tier: Option<&'a str>,
  #[serde(skip_serializing_if = "Option::is_none")]
  pub prompt_cache_key: Option<&'a str>,
  ```
  Both `Option<&'a str>` borrowed and `skip_serializing_if = "Option::is_none"` — payload backward-compatible (fields absent on legacy callers, server treats as None).
- **Auth-mode gating** at `core/src/client.rs:442-450`:
  ```rust
  let compact_service_tier = if client_setup
      .auth
      .as_ref()
      .is_some_and(CodexAuth::is_chatgpt_auth)
  {
      settings.service_tier
  } else {
      None
  };
  ```
  Correct: `service_tier` is a ChatGPT-auth concept (Fast→priority), so omitting it for API-key auth avoids a 400 from the server-side schema. The asymmetric handling for `prompt_cache_key` (always passed) is also correct — caching is universal.
- **Request-builder reuse** at `client.rs:454-462,475-481`: pulls `service_tier` and `prompt_cache_key` out of the *normal* `build_responses_request(...)` output and re-injects them into `CompactionInput`. This is the right pattern — guarantees the compact payload's fields stay in lockstep with the sampling payload's fields without duplicating the resolution logic.
- **Visibility tightening**: `compact_conversation_history` changes from `pub async fn` at `:413` to `pub(crate) async fn` at `:418` — confirms there are no external callers and the new `CompactConversationRequestSettings` struct stays crate-private.
- **`CompactConversationRequestSettings`** at `client.rs:148-153` bundles the three previously-loose params (`effort`, `summary`, `service_tier`) — call-site readability win, even before the third field is added.
- **Insta snapshot tests** at `core/tests/suite/compact_remote.rs` (+279 lines, two snapshots):
  - `remote_manual_compact_api_auth_prompt_cache_key_request_diff.snap` (+44 lines): asserts `prompt_cache_key` is reused, `service_tier` omitted under API-key auth.
  - `remote_manual_compact_chatgpt_auth_service_tier_prompt_cache_key_request_diff.snap` (+43 lines): both reused under ChatGPT auth.
  - Driven through five varied turns (plain text, multi-part text, tool-call continuation, image+text, local-shell continuation, final-turn reasoning) — coverage of the request-builder paths.
- **Test scaffolding** at `core/tests/common/context_snapshot.rs` (+163 lines, new file) — diff-snapshot helper, reusable for future request-parity tests.
- **`tui/src/resume_picker.rs:0/-3`** — three-line subtractive cleanup, separate from the main change but consistent with reduced surface area.

## Nits

1. **PR explicitly says "Not run locally per repo guidance"**. Insta snapshot tests are deterministic so CI alone is reasonable, but a maintainer should confirm CI green-checked the two new `.snap` files before merge — insta diffs are easy to mis-accept locally.
2. **No `Fast`-without-ChatGPT regression test**: the PR adds a snapshot for `Fast` *with* ChatGPT auth (where it should map to `priority`) and a snapshot for API-key auth (where service_tier omits), but no explicit assertion that `Fast` configured under API-key auth doesn't accidentally leak through. The current `is_some_and(CodexAuth::is_chatgpt_auth)` check should prevent it, but a dedicated snapshot would harden the gate.
3. **`prompt_cache_key` is universal** — worth a one-line comment at `:451-453` documenting the asymmetric rationale (cache key is provider-agnostic, service_tier is ChatGPT-only) for future maintainers.
4. The new `CompactConversationRequestSettings` struct is `pub(crate)` — fine, but if any downstream crate in `codex-rs/` wants to drive compaction directly the change is a breaking API surface. Worth a quick `rg 'compact_conversation_history' codex-rs/` to confirm only `compact_remote.rs` calls it.

Surgical, well-scoped, snapshots cover both auth modes. Merge after CI green.
