# sst/opencode #25805 — fix(opencode): add max_retries config to cap session retry attempts

- **Repo**: sst/opencode
- **PR**: [#25805](https://github.com/sst/opencode/pull/25805)
- **Author**: Fatty911
- **Head SHA**: `2aec720f95b68c5cd30b8f036348f31c73e760a0`
- **Base branch**: `dev`
- **Created**: 2026-05-05T02:16:07Z
- **Closes**: #25733

## Verdict

`merge-after-nits`

## Summary of change

Adds an optional `experimental.max_retries` (PositiveInt) knob to the session
config so users can cap retry attempts on transient provider errors instead of
the current "retry forever" behavior. When unset, behavior is unchanged.

Three small surface points:

- `packages/opencode/src/config/config.ts:260-263` — schema adds the new optional
  field with a clear description that includes the unbounded default.
- `packages/opencode/src/session/processor.ts:678` — reads
  `(yield* config.get()).experimental?.max_retries` once per outer turn and
  forwards it to `SessionRetry.policy({ ..., maxRetries })` on line 705.
- `packages/opencode/src/session/retry.ts:106-115` — `policy()` accepts the
  optional `maxRetries`; the early-exit check
  `if (opts.maxRetries !== undefined && meta.attempt >= opts.maxRetries) return Cause.done(meta.attempt)`
  fires before the delay/wait branch.

## What's good

- Correctly additive: existing call sites without the field keep "retry forever".
- The check uses `>=` against `meta.attempt`, so `max_retries: 1` means "one
  retry then surface the error" (matches what most users would expect from a
  config named `max_retries`).
- Schema description explicitly states "When not set, retries are unbounded",
  which is the right thing to call out for an open-source default.
- Lives under `experimental.*` so the API surface is honestly framed as
  unstable.

## Nits / questions

- **Off-by-one semantic ambiguity** (`retry.ts:113`): `meta.attempt` from
  `Schedule.fromStepWithMetadata` is 0-indexed for the *next* attempt the
  schedule is being asked about. Worth a one-line comment clarifying whether
  `max_retries: N` = "N retries after the first failure" (so N+1 total
  provider calls) or "N total attempts including the first" — the description
  on the schema field says "Maximum retry attempts" which leans toward the
  former, but a contributor reading just `retry.ts` won't know. Add either a
  test or a docstring example: `max_retries: 3` → 1 initial + 3 retries = 4
  provider calls.
- **No test for the new branch**. The retry policy is testable in isolation
  (the schedule is pure given a parser and a clock); a small unit test that
  feeds N retryable errors and asserts the schedule terminates after exactly
  N+1 (or N, depending on the chosen semantic) would lock the contract in.
- **`max_retries: 0` edge case**: `PositiveInt` presumably forbids 0, but the
  description "Maximum retry attempts ... before stopping" is ambiguous about
  whether 0 means "never retry" or is rejected by the schema. Consider stating
  in the description that the minimum is 1, or accept 0 with the meaning
  "fail on first retryable error".
- The `set` callback on line 32 of `retry.ts` (the `EventV2.run(SessionEvent.Retried.Sync, ...)`
  invocation in `processor.ts:705-710` — visible in the diff hunk at
  `processor.ts:702-707`) still fires on the attempts that *do* retry; once the
  cap is hit, no event is emitted to indicate "retry budget exhausted". A
  single terminal event (e.g. `RetryBudgetExhausted`) would help downstream
  observability.
- `processor.ts:678` resolves `maxRetries` once at the start of the outer
  Effect; that's correct under the current model where config is read once per
  turn, but worth confirming no hot-reload path invalidates that assumption.

## Risk

Low. The change is additive, gated on `opts.maxRetries !== undefined`, and the
default path through `policy()` is unchanged byte-for-byte. The biggest risk is
documentation drift if the semantic of `meta.attempt` is reinterpreted later.

## Recommendation

Land after (a) one unit test covering the cap path and (b) a one-line
docstring on the schema description making the "N retries beyond the first
attempt" semantic unambiguous.
