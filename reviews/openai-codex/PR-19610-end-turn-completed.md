# openai/codex PR #19610 — Support end_turn in response.completed

- **PR**: https://github.com/openai/codex/pull/19610
- **Author**: @andmis (Andrey Mishchenko)
- **Head SHA**: `a21212b0dda597e01a417536dc23fd2d43124f17`
- **Size**: +26 / −5

## Summary

Adds an optional `end_turn: Option<bool>` field to `ResponseEvent::Completed` and the corresponding SSE `ResponseCompleted` deserialiser, then threads it from the API layer down through `core::client` and `session::turn`. When the model affirmatively signals `end_turn=false`, `try_run_sampling_request` flips `needs_follow_up` so the session loop will keep going. When the field is absent (older provider responses, or providers that never set it) the existing fallback heuristics still apply.

## Verdict: `request-changes` (small but blocking)

The substantive change is correct, but the CLI's `responses_cmd.rs:81` introduces a `// FIXME(andrey): Make this actually work + test it, before merging.` next to the destructured `end_turn: _`. That comment is the author flagging the JSON output path as deliberately unfinished, and the destructure literally throws the new signal away in the only command that surfaces SSE events to humans for debugging. This is the one place where regressions on `end_turn` would be invisible until production. The right move is to either propagate `end_turn` into the emitted JSON (it's just another field on the response object), or drop the FIXME and add a one-line test that asserts the field is *intentionally* not exposed yet.

Everything else (`codex-api/src/common.rs`, `sse/responses.rs`, `core/src/client.rs`, `core/src/session/turn.rs`) looks right.

## Specific references

- `codex-rs/cli/src/responses_cmd.rs:81-82` — `// FIXME(andrey): Make this actually work + test it, before merging.` followed by `end_turn: _,`. Block-merge IMO until the FIXME is resolved one way or the other; the existing test `assert_eq!(completed, …)` at `:172` and `:195` only checks `response_id` + `token_usage` so the field's absence in JSON isn't even asserted as deliberate.
- `codex-rs/codex-api/src/common.rs:84` — the new field with a clear docstring (`Did the model affirmatively end its turn? Some providers do not set this, so we rely on fallback logic when this is None.`). Good.
- `codex-rs/codex-api/src/sse/responses.rs:126` — `end_turn: Option<bool>` with `#[serde(default)]`. Correct: missing field deserialises to `None`, preserving back-compat with providers that don't emit it.
- `codex-rs/core/src/session/turn.rs:2147-2149` — `if let Some(end_turn) = end_turn { needs_follow_up |= !end_turn; }`. The `|=` is important: an upstream signal that the turn is *not* done can promote a `needs_follow_up=false` decision to `true`, but a signal that the turn *is* done cannot suppress a follow-up that other heuristics already requested. That's the conservative direction — good.
- `codex-rs/Cargo.lock:2873` — the `codex-utils-absolute-path` dependency is removed from one crate's lockfile entry. Looks like a tangential clean-up that snuck into this PR; verify it's intentional and not a stale rebase artefact.

## Nits

1. Resolve the FIXME, either by surfacing `end_turn` in the JSON or by adding a focused test that locks in the deliberate omission.
2. The `Cargo.lock` removal of `codex-utils-absolute-path` should be in its own commit (or PR) so it doesn't muddy the history of the `end_turn` plumbing.
3. Consider one positive-path integration test in `sse_end_to_end.rs` where the SSE payload contains `end_turn: false` and the resulting `ResponseEvent::Completed` carries `end_turn: Some(false)` — currently every test in this PR asserts only the `None` branch.
