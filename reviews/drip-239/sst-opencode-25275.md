# sst/opencode#25275 — fix(session): use finite archived timestamp schema

- **PR**: https://github.com/sst/opencode/pull/25275
- **Author**: kitlangton
- **Head SHA**: `cb8f181fd33cfa8bcf694a6cc64f3f1caea39166`
- **Files**: `packages/opencode/src/session/session.ts`, `packages/opencode/test/server/httpapi-bridge.test.ts`
- **Verdict**: **merge-after-nits**

## Context

`ArchivedTimestamp` at `session.ts:142-145` (pre-fix) was declared as `Schema.Number`, which under the Effect Schema → OpenAPI generator widens to a `number | "Infinity" | "-Infinity" | "NaN"` union (string-literal arms for the non-finite values). That union surfaces in the generated SDK types as a `number | string` field, which broke any consumer that typed `archived` as a plain `number` in their generated client. The fix flips the schema to `Schema.Finite` so the OpenAPI emitter produces a clean `{type: "number"}` shape.

## What's right

- **The schema change at `session.ts:144`** (`export const ArchivedTimestamp = Schema.Finite`) — `Schema.Finite` is the correct effect-schema primitive for "any IEEE-754 number except `±Infinity` and `NaN`". This is exactly the contract the JSON-round-trip requires (JSON has no representation for non-finites; `JSON.stringify(Infinity) === "null"`), so the OpenAPI surface narrowing is a *correctness* fix, not just a type cosmetic.
- **The legacy negative-value case is preserved** by the comment at `:142-143` (`"Legacy HTTP accepted negative values here. Keep archive timestamps permissive while excluding non-finite values that cannot round-trip through JSON"`). Earlier persisted data with `archived: -1` (the documented "unarchived" sentinel from earlier versions) still validates because `Finite` doesn't exclude negatives — only the structural `Infinity`/`NaN`/`-Infinity` arms.
- **The bridge test at `httpapi-bridge.test.ts:261-272`** is the right shape of regression test: it walks `effectOpenApi().paths["/session/{sessionID}"]?.patch?.requestBody.content["application/json"].schema.properties.time.properties.archived` and asserts `.toEqual({type: "number"})`. This pins the *generated SDK shape*, not just the runtime validator behavior — i.e. the regression closes the actual user-visible bug (the SDK type drift) rather than merely the underlying schema.

## Risks / nits

- **No test pins the actual `Schema.Finite` rejection behavior.** The bridge test only asserts the generated OpenAPI shape; it doesn't assert that `Schema.decodeSync(SessionUpdateRequest)({time: {archived: Infinity}})` actually throws (it should, given `Finite`). Worth one more test arm so a future schema-library upgrade that silently drops `Finite`'s validation can't pass without notice.
- **No documentation/migration note for callers who *were* persisting `±Infinity` as a "never archived" sentinel.** Probably none exist (the schema was just `Number` so passing `Infinity` would have been a JSON-serialization error end-to-end anyway), but a one-line PR-body or CHANGELOG mention that the validator now actively rejects non-finites would close the "did anyone depend on this?" question.
- **The same generator-emit-string-literal issue likely exists for other `Schema.Number` fields in the codebase.** A grep of `Schema.Number` across `session.ts` would catch other timestamp/duration-shaped fields that should also be `Finite` (or `NonNegativeInt` like the sibling `Time.created` field at `:148`). Not blocking for this PR but worth a follow-up sweep.

## Verdict

**merge-after-nits.** The fix is a one-line correctness change with the right primitive (`Schema.Finite`), and the test pins the user-visible OpenAPI shape regression. The only real ask is one more test arm pinning the rejection behavior so the schema-validation contract is locked alongside the schema-emission contract.
