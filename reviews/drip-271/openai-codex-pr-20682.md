# openai/codex #20682 — feat(app-server): always return limited thread history

- URL: https://github.com/openai/codex/pull/20682
- Head SHA: `9e7312ab1c12f019dd309c6def36afe336796ea2`
- Author: @owenlin0
- Stats: medium — touches `v2.rs` protocol docs, `bespoke_event_handling.rs`,
  and `codex_message_processor.rs`

## Summary

Inverts the meaning of `persistExtendedHistory`: it still controls
**what gets persisted** to the rollout file, but `thread/read`,
`thread/resume`, and `thread/fork` now uniformly return only the
"limited" (api) form of thread history regardless of the flag. The
read path swaps `build_turns_from_rollout_items` →
`build_api_turns_from_rollout_items` at every call-site, and adds a
`ThreadHistoryBuilder` import for the new abstraction.

## Specific feedback

- `codex-rs/app-server-protocol/src/protocol/v2.rs:3614-3617,
  3746-3749, 3852-3855` — three identical doc-block updates on
  `ThreadStartParams`, `ThreadResumeParams`, and `ThreadForkParams`.
  All three correctly call out the asymmetry: "persists more, returns
  the same limited subset for scalability." Critical clarification —
  before this PR, the docs implied a richer return type.
- `codex-rs/app-server/README.md:307-308` — removes the "experimental
  API: persistExtendedHistory" section entirely. Probably right, given
  the new semantics make the flag a write-only knob, but consider
  replacing with a single line documenting the persist-vs-return split
  so downstream IDE clients know what they're getting.
- `codex-rs/app-server/src/bespoke_event_handling.rs:1180` and
  `codex_message_processor.rs:4044, 4109` — three call-sites updated
  in lockstep. The grep for `build_turns_from_rollout_items` should
  now have zero results in this directory; please verify before merge.
- The behavioural change is **scalability-driven** ("for scalability
  reasons" appears verbatim in three doc blocks), implying the full-
  history return path was OOM-ing or saturating IPC for large threads.
  Worth a one-line `// TODO: revisit if a paginated full-history API
  lands` comment so this isn't taken as the final state.
- Backward-compat risk: any client that opted into
  `persistExtendedHistory: true` expecting `thread/read` to give it
  the richer items is now silently downgraded. The protocol field
  remains `#[experimental]`, so the breakage is "permitted", but
  release notes should call this out explicitly.
- `EventPersistenceMode` and `is_persisted_rollout_item` are imported
  in `codex_message_processor.rs:376-377` but I didn't see their first
  use in this slice — confirm they're consumed (lints will catch
  otherwise).
- No new tests. The semantic flip "persistFullHistory now means
  persist-only, not return-fully" deserves at least one integration
  test asserting that `persistExtendedHistory: true` + `thread/read`
  returns the limited shape.

## Verdict

`merge-after-nits` — the docs change is necessary (current docs are a
landmine) and the call-site swap is mechanical. Add the regression
test and a README line documenting the persist-vs-return split before
merge.
