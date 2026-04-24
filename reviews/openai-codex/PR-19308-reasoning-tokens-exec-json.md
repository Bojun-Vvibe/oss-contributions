# PR #19308 — surface reasoning tokens in `codex exec --json` usage

**Repo:** openai/codex • **Author:** etraut-openai • **Status:** merged
• **Net:** +9 / −1

## Change

Adds `reasoning_output_tokens: i64` to the public `Usage` struct in
`codex-rs/exec/src/exec_events.rs` and wires it through the JSONL
event processor by reading `usage.total.reasoning_output_tokens` from
the existing `ThreadTokenUsageUpdated` payload. The TypeScript SDK
event type `Usage` and two test fixtures (`run.test.ts`,
`runStreamed.test.ts`) get the matching field.

## Why this matters

`codex exec --json` is the only stable surface for programmatic
consumers that don't write rollout files (CI runners, scripted
evaluations, pre-commit agents). They already received `input_tokens`,
`cached_input_tokens`, and `output_tokens` on `turn.completed.usage`,
but the reasoning split was silently dropped — even though the same
data was already populated upstream and exposed through the rollout
path. For reasoning-model billing analysis, that meant exec consumers
saw `output_tokens` as a single bucket and had to back-derive
reasoning vs visible output by diffing against API receipts.

## Risk: silent breakage of strict consumers

The new field is added without a default in either the Rust struct
(it's `i64`, not `Option<i64>`) or the TypeScript type. Any existing
consumer that:

1. Deserializes `Usage` with `serde` plus `deny_unknown_fields` (none
   in-tree, but downstream wrappers commonly add it), or
2. Has a TypeScript schema validator (`zod`, `io-ts`) with strict
   object shape,

will now reject prior-format events emitted by older `codex exec`
binaries — the reverse compat direction. This isn't a bug per se, but
the PR description doesn't flag the SDK type as a breaking field
addition. A `?:` (optional) on the TS side would have been the safer
path; it's a breaking-ish bump even though `0` is a sensible default.

## Risk: the `i64` choice locks in an upstream constraint

`ThreadTokenUsageUpdated` already exposes `reasoning_output_tokens`
as an `i64`; the PR mirrors that. But every other token field in this
struct is also `i64`, and the entire payload is regularly summed by
consumers. Reasoning tokens for some recent reasoning models can
exceed `2³¹` over a long-running session — `i64` is fine, but if a
consumer downcasts to `i32` (older Java/Go SDKs) the sum overflows
silently. Worth a one-line note in `events.ts` recommending sum-side
arithmetic in 64-bit.

## Concrete next step

Add `reasoning_output_tokens: 0` as the default in any future
`Usage` deserialization site (use `#[serde(default)]` on the field
in the Rust struct), and mark the TS field as `reasoning_output_tokens?:
number` plus a one-liner in the SDK CHANGELOG that pre-19308 binaries
will now produce events without this field — current code path makes
that fail validation in strict consumers. Also: surface the
`reasoning_output_tokens / output_tokens` ratio in `basic_streaming.ts`
demo, since it's the single most-requested signal for cost dashboards.

## Verdict

Correct, minimal fix for a real observability gap. The only sharp
edge is the backward-compat asymmetry — old binary, new consumer is
fine; new binary, old strict consumer is fine; new consumer reading
old binary output will fail. Worth a release note.

## What I learned

When public JSONL surfaces grow a field, the breaking direction is
"old producer + new strict consumer", not the other way around.
Defaulting on the consumer side via `#[serde(default)]` / `?:`
costs nothing and removes the only real foot-gun.
