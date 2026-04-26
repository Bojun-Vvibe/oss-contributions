# Review — sst/opencode#24512: Refactor v2 session events as schemas

- **Repo:** sst/opencode
- **PR:** [#24512](https://github.com/sst/opencode/pull/24512)
- **Head SHA:** `9def97f99b9aa0cf33fa899c424b4be4a4dc47c6`
- **Branch:** `refactor/session-event-schema-definitions`
- **Size:** +719 / -1087 across 7 files (net −368)
- **Verdict:** `merge-after-nits`

## Summary

Reworks the v2 session event surface from class-based schemas into `Event.define(...)`-style const definitions, renames every event type from terse names (`prompt`, `synthetic`, `step.started`, `text.delta`) to fully-qualified `session.*` types (`session.prompted`, `session.synthetic`, `session.step.started`, `session.text.delta`), introduces a shared `packages/opencode/src/v2/event.ts:1-41` generic `define()` helper that simultaneously produces an Effect Schema struct and a `SyncEvent.define`-wrapped sync envelope, extracts prompt-attachment schemas into `session-prompt.ts` (+36), and rewrites `test/session/session-entry-stepper.test.ts` from a 676-line FastCheck + custom-adapter property test suite into a 401-line deterministic memory-reducer set (net −275 in that file alone).

## Technical assessment

The `Event.define()` generic at `event.ts:13-37` is the right shape: it takes a literal `type`, an Effect-Schema `Fields` map, an `aggregate` namespace, and an optional `version` (defaulting to 1), then returns a `Schema.Struct` with the canonical event envelope (`id`, `metadata` optional, `timestamp`, `type` literal, `version` optional, plus user fields) annotated with `identifier: input.type` for serialization, plus a `Sync` property carrying the `SyncEvent.define` wrapper. The `Object.assign(Event, { Sync })` pattern at `:35` keeps both addressable from one import and matches Effect-ecosystem conventions. `ID` at `:6-11` correctly brands `evt`-prefixed identifiers with `withStatics` for `.create()` ergonomics — same pattern as the rest of the v2 ID family.

The renames at `session-entry-stepper.ts:67,75,78,88,99,111` are mechanical and the `SessionEvent.Event.match` discriminator preserves exhaustiveness because the `Schema.Literal(input.type)` at `event.ts:23` keeps `type` narrowable. Test rewrite shifts from "generate arbitrary event sequences and assert reducer invariants" to "deterministic memory reducer cases" — that's a real loss of coverage for property-based edge cases (the FastCheck suite at the −676 lines was finding ordering bugs that hand-rolled cases miss), but a real win for failure diagnosability since FastCheck shrinking on a session-event reducer was probably eating CI minutes.

## Nits worth addressing pre-merge

1. **Breaking change is not gated.** The event type rename `prompt` → `session.prompted`, `step.started` → `session.step.started`, etc. is a wire-protocol breaking change for any v2 client that has persisted events with the old `type` strings (sync replay, on-disk session journals, plugin event listeners). The PR description says "Refactor v2 session events as schemas" but doesn't acknowledge the breaking surface. Either (a) confirm that v2 is not yet shipped/persisted to disk anywhere a real user has, (b) add a one-shot migration in the sync replay path that aliases old → new types on read, or (c) bump a `SCHEMA_VERSION` constant somewhere callers can branch on.

2. **`metadata` and `version` both optional.** At `event.ts:21,25` both `metadata` and `version` are `Schema.optional`. That's fine for ingest but means downstream consumers that want to index by version need to default-handle `undefined`. Consider `Schema.optionalWith(Schema.Number, { default: () => 1 })` so the runtime always has the value, or document the expected contract.

3. **`Schema.Record(Schema.String, Schema.Unknown)`.** The metadata at `:21` is `Record<string, unknown>` which is the right escape hatch but means typos in metadata keys silently flow through. Not blocking — just note that any future "well-known metadata keys" should get their own typed sub-schema rather than living in this bag.

4. **Test file parity.** Net −275 in `session-entry-stepper.test.ts` is a meaningful coverage delta. Confirm via `bun test --coverage test/session/session-entry-stepper.test.ts` that branch coverage of `session-entry-stepper.ts` did not regress versus the FastCheck baseline. If it did, restore the most valuable property test as a deterministic case (the "interleaved step.started / text.delta with mid-stream cancellation" scenario was the historical bug-finder).

5. **`session-prompt.ts` as a new file.** +36 lines extracting prompt attachment schemas is a clean win, but it's not clear from the diff whether anything still defines prompt schemas inline in `session-event.ts`. Grep for `Schema.Struct` blocks describing attachments and consolidate.

## Verdict rationale

`merge-after-nits` because the refactor pattern is right and the diff direction (−368 net) is healthy, but the wire-format rename without a version bump or migration note is the kind of thing that bites a downstream sync consumer six weeks later. Address the breaking-change story (acknowledge in PR body + CHANGELOG, or add the alias) and the FastCheck coverage spot-check, then merge.
