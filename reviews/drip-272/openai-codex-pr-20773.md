# openai/codex PR #20773 — feat: implement remote compaction v2 client path

- URL: https://github.com/openai/codex/pull/20773
- Head SHA: `d86457398018c919da3c7acf9c79552bd6473dda`
- Verdict: **needs-discussion**

## What it does

Introduces a new `ContextCompaction` `ResponseItem` variant alongside the
existing `Compaction` variant, gated by a new `remote_compaction_v2` feature
flag, and wires the new variant through every `ResponseItem` match arm in
the core crate. Also adds beta-features header propagation in `client.rs`
so the server knows the client supports the v2 protocol.

## Specifics I checked

- New variant in `ResponseItem.ts` schema (line 17) with optional
  `encrypted_content` (vs. `Compaction`'s required `encrypted_content`).
  Mirrored in 5 `*.schemas.json` files.
- `codex-rs/core/src/agent/control.rs:115-118`,
  `arc_monitor.rs:384-387`,
  `compact.rs:420-426` (truncated in diff) all add the new variant to
  exhaustive `match` arms, which is exactly the kind of change the
  Rust compiler enforces — a structurally safe rollout.
- `client.rs:471-477` adds an unconditional
  `extra_headers.extend(build_responses_headers(self.state.beta_features_header.as_deref(), None, None))`
  call before the existing identity-headers extension. **This sends the
  beta features header on every Responses API request, not only when
  `remote_compaction_v2` is enabled.**
- New feature property `remote_compaction_v2: bool` in two locations of
  `config.schema.json` (per-profile + root features).

## Concerns

1. **The header is sent unconditionally.** The new
   `build_responses_headers` call in `client.rs:471` doesn't check whether
   the feature is on. If `state.beta_features_header` is `Some` for any
   reason, every request now leaks the beta features list to the server
   regardless of whether `remote_compaction_v2` is the only opted-in beta.
   Confirm this is intentional — the typical pattern would be to gate this
   behind `if features.remote_compaction_v2 { ... }`.
2. **Two compaction variants in parallel forever?** `Compaction` (required
   `encrypted_content`) and `ContextCompaction` (optional `encrypted_content`)
   coexist in the same `ResponseItem` enum. The end-state after rollout
   isn't documented in the PR. If v2 obsoletes v1, you'll need a migration
   plan for serialized rollouts on disk; if not, the schema-level
   distinction needs a short doc comment so future readers know which to
   produce.
3. **`encrypted_content` is optional in v2** but the PR doesn't say what it
   means when absent — does the server compute compaction server-side and
   the client just records the marker? Or is the client expected to fill it
   in later? This affects whether downstream consumers (forking, replay
   in `agent/control.rs`'s `keep_forked_rollout_item`) should treat an
   absent `encrypted_content` as "drop" vs. "keep marker but no payload".
4. **PR body is empty.** Same complaint as #20779.

## What I like

- Exhaustive match coverage: every `match item { ResponseItem::... }` arm
  has been updated. Good Rust discipline.
- New variant rather than mutating `Compaction` — preserves on-disk
  rollout compatibility for existing sessions.

## Verdict rationale

The mechanical wiring is correct, but the unconditional beta-header send
plus the underspecified semantics of optional `encrypted_content` warrant
discussion before merge. Not a "request-changes" because the design may
well be intentional and just needs documentation; bumping to
`needs-discussion`.
