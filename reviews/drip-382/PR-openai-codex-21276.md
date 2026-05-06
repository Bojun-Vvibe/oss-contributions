# openai/codex PR #21276 — [codex] Remove unused ListModels op

- URL: https://github.com/openai/codex/pull/21276
- Head SHA: `4ef3a72bbc418b42e72eefcacb692696e24caaf7`
- Size: +0 / -4

## Summary

Strict subtractive refactor of the core submission `Op` enum. The
`Op::ListModels` variant in `codex-rs/protocol/src/protocol.rs` was already
dead code — no client emitted it, the core submission loop treated it as
unknown, and the actual model-list functionality lives behind the app-server
JSON-RPC `model/list` endpoint. PR deletes the variant + its `Op::kind()`
mapping, leaving the wire-protocol surface honest about what's actually
supported.

## Specific findings

- `codex-rs/protocol/src/protocol.rs:799-801` (deleted) — removes the
  `/// Request the list of available models.\n    ListModels,` arm. Variant
  carries no fields, so removal can't break payload-shaped serde consumers.
- `codex-rs/protocol/src/protocol.rs:904` (deleted) —
  `Self::ListModels => "list_models",` arm dropped from `Op::kind()`. Match
  remains exhaustive after the variant deletion; no `_` catch-all was added,
  which is the correct way to maintain compile-time exhaustiveness across
  future variant additions.
- The PR description correctly identifies app-server `model/list` as the
  active surface — that endpoint is unrelated to this enum and is unchanged.
- Net diff is exactly 4 deletions across one file, making blast-radius
  trivially auditable.

## Notes

- Because `Op` is a `#[derive(Serialize, Deserialize)]` enum with the
  default external tagging strategy (not `#[serde(untagged)]` or
  `#[serde(other)]`), an in-flight client that *did* send
  `{"type":"ListModels"}` would now hit a deserialization error rather than
  silently no-op. PR description asserts no client sends this op; if any
  external integration / SDK had been emitting it, this is a (minor)
  breaking wire change. Worth a one-line maintainer confirmation that the
  audit covers all known SDK languages (TS bindings, Python wrappers, etc.)
  before merge.
- Verification is `cargo test -p codex-protocol` only. A targeted
  `cargo check -p codex-core` to confirm no internal call site still pattern-
  matches `Op::ListModels` would harden it; `rg 'Op::ListModels' codex-rs/`
  should return zero hits.
- No `CHANGELOG` / migration note. For a dead-code variant this is fine, but
  if there's a public protocol-changelog convention in this repo a one-line
  entry would be appropriate.

## Verdict

`merge-after-nits`
