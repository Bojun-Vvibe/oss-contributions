---
pr: sst/opencode#25011
sha: bc2a3a91b734cd5d9616e936fde3346deef81429
verdict: merge-after-nits
reviewed_at: 2026-04-30T00:00:00Z
---

# fix: use moonshot MFJS sanitization to prevent api errors w/ kimi models

URL: https://github.com/sst/opencode/pull/25011
Files: `packages/opencode/src/provider/transform.ts`, `packages/opencode/test/provider/transform.test.ts`

## Context

The Moonshot/Kimi MFJS sanitizer at `transform.ts:1089` previously rebuilt the
schema immutably via `Object.fromEntries`. That worked for the `$ref` sibling
case it was originally added for, but Kimi's MFJS validator also rejects
`prefixItems`, `unevaluatedItems`, `exclusiveMinimum`/`exclusiveMaximum`,
`minContains`/`maxContains`, and the annotation fields `title`, `$comment`,
`format`. Tools whose JSON-schema came from libraries that emit any of those
keys (Pydantic, zod-to-json-schema with strict mode, `int32`-formatted
integers) would fail with 400 from the API.

## What changed

`transform.ts:1089-1119` flips the helper from immutable (return new object) to
in-place (`void`). For each node:

- `$ref` siblings are stripped via `delete` keys (lines 1098-1102) instead of
  reconstructing `{ $ref }`.
- Annotation keys `title`, `$comment`, `format` and complex validation keys
  `exclusiveMinimum`, `exclusiveMaximum`, `minContains`, `maxContains` are
  unconditionally deleted (lines 1103-1108).
- `prefixItems` is captured then deleted; if `items` isn't already an object,
  the first `prefixItems` entry becomes `items` (lines 1109-1118). This
  collapses tuple arrays into the single-schema form MFJS requires.
- `unevaluatedItems` is dropped.

Tests at `transform.test.ts:1000-1073` cover all three new branches:
prefixItems collapse, annotation stripping (preserving `description`/`default`),
and complex-validation stripping (preserving `minimum`/`contains`).

## Risks

- **In-place mutation of caller schema.** The signature changed from
  `(obj) => obj` to `(obj) => void`, and the function now mutates the
  `schema` argument directly. Callers that hold a reference to the original
  schema (e.g. caching layers, telemetry) will observe their copy mutated.
  The diff at `:1121` (`sanitizeMoonshot(schema)`) suggests the caller
  passes the model's tool schema by reference, which means a subsequent
  call for a non-Moonshot model in the same process could see the already-
  stripped schema. Worth confirming this is only invoked on a freshly built
  per-request schema.
- The `$ref` branch dropped the `typeof obj.$ref === "string"` *guard inside
  an `in` check* in favor of just `typeof obj.$ref === "string"` — fine
  semantically, but means a sibling-only object with no `$ref` will fall
  through to the deletion loop, which is the intended behavior.

## Verdict

`merge-after-nits` — the sanitization expansion is correct and well-tested for
the documented MFJS constraints, but the switch to in-place mutation is a
subtle contract change worth a one-line comment ("mutates input") above the
helper, and ideally a sanity check that `provider/transform.ts` callers don't
share the schema object across providers.
