# sst/opencode #24716 — fix(httpapi): align sync seq validation

- **Head SHA**: `5ae2ba3d92ae9162397542736d9ade55cad86604`
- **State**: OPEN
- **Author**: kitlangton
- **Size**: +52 / -5 across 2 files

## Files changed

- `packages/opencode/src/server/routes/instance/httpapi/sync.ts` (+6/-3)
- `packages/opencode/test/server/httpapi-sync.test.ts` (+46/-2)

## What it does

Closes a real validation-parity gap between the legacy Hono `/sync` router (zod) and the experimental Effect HttpApi router (Schema), where the latter's `seq` field used a bare `Schema.Number` and silently accepted negative integers and fractional values that the legacy zod schema rejected at the door. Two surfaces, two parsers, divergent acceptance set — exactly the kind of drift that produces "works in legacy, mysteriously fails in experimental" bug reports once a downstream consumer relies on the rejection.

## Specific observations

- **The fix is a single named schema brick**, `Seq = Schema.Int.check(Schema.isGreaterThanOrEqualTo(0))` at `packages/opencode/src/server/routes/instance/httpapi/sync.ts:16`, then reused at the two leaf positions that previously used `Schema.Number`: the `seq` field of `ReplayEvent` (`:21`) and the value type of `HistoryPayload`'s `Schema.Record(...)` (`:32`). Naming the constraint once and reusing it is the right call — copy-pasted inline `Schema.Int.check(...)` in both places would invite drift on a future "actually we want strictly-positive" tightening.
- **`Schema.Int` is the load-bearing piece**, not the `>= 0` check. `Schema.Number` accepts `1.5`; `Schema.Int` rejects it as a separate failure class before the range check runs. The two failure modes are semantically distinct (non-integer vs. negative) and Effect Schema will surface them differently in error messages, which is the right shape — a future API consumer debugging "why did my `seq: 1.5` get rejected" will see "expected Int" not "expected number ≥ 0".
- **Adding `error: HttpApiError.BadRequest` to the two endpoints at `:63` and `:74`** is what actually makes the validation failure produce a structured 400 instead of a generic 500. Without this annotation the Effect HttpApi router would have caught the schema decode failure but had no declared error variant to map it to, defaulting to 500 — the test would still have been red, but for the wrong reason. Both endpoints get the same treatment which is consistent with the schema living in a shared module.
- **The parity test at `test/server/httpapi-sync.test.ts:84-127` is the right spec.** Helper `app(httpapi = true)` parameterizes on the same `OPENCODE_EXPERIMENTAL_HTTPAPI` flag the production code reads (`:19-22`), so each loop iteration constructs both surfaces from the same source of truth. The 4-cell fixture matrix is `(history, replay) × (negative, fractional)` — exactly the cells the schema change targets; a future "rejects strings too" tightening would need a new cell, but for the current fix this is complete. The two final asserts at `:122-125` (`httpapi.status === legacy.status` then `=== 400`) lock down both the parity invariant *and* the absolute expectation — without the second assert a future regression that made *both* surfaces accept `-1` would still pass parity and silently weaken validation everywhere.
- **Loop iteration order risk**: the `for (const item of cases)` at `:111` mutates `Flag.OPENCODE_EXPERIMENTAL_HTTPAPI` twice per iteration via `app(false)` then `app(true)`. This is the same pattern the existing test at `:6` (`originalHttpApi`/`originalWorkspaces`) uses for restore, so consistent with conventions in the file, but a parallel test runner (vitest with `--threads`) would race against the module-global flag. Probably out of scope for this PR (existing tests have the same exposure) but worth noting if anyone considers parallelizing this suite.
- **`Schema.Record(Schema.String, Seq)` at `:32`** is a type-narrow on every value in the history payload map. Worth confirming that `Schema.Record` performs per-value validation and surfaces the bad key path in the error (so a payload `{aggregate: -1, other: 5}` produces a useful error pointing at `aggregate`). The test fixture only uses single-key records (`{aggregate: -1}`) so multi-key error-path quality is unverified.
- **Missing positive case**: the new test only covers rejection. A `seq: 0` and `seq: 42` accept-path test would pin the inclusive lower-bound semantic (`isGreaterThanOrEqualTo(0)` includes 0) — the existing happy-path test at `:55-83` already exercises some `seq` values but with the Effect path before this change accepting everything, the test was tautological for the validation contract. With validation now active, a positive boundary test would catch a future mistype to `isGreaterThan(0)`.

## Verdict

**merge-after-nits** — surgical parity fix with the right named-schema-brick shape, correct add of `HttpApiError.BadRequest` declarations on both endpoints to make validation failures produce 400s instead of 500s, and a parity test matrix that pins both legacy/experimental equality and the absolute 400 expectation against future drift. Nits before merge: a positive-boundary test for `seq: 0` to pin inclusive lower-bound semantics against a future mistype to `isGreaterThan(0)`, and ideally a multi-key `HistoryPayload` rejection case to verify `Schema.Record` surfaces useful per-key error paths.
