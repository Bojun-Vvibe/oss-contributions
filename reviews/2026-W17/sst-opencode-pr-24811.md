# sst/opencode PR #24811 — fix(httpapi): align request body openapi shape

- URL: https://github.com/sst/opencode/pull/24811
- Head SHA: f2e666f633ee
- Files: `packages/opencode/src/server/routes/instance/httpapi/public.ts`, `packages/opencode/test/server/httpapi-bridge.test.ts`
- Verdict: **merge-after-nits**

## Context

The Effect HttpApi-generated OpenAPI spec and the legacy hand-rolled
spec disagree on request-body shape: the Effect side emits richer
schemas with `anyOf`/`oneOf`/null-unions, while the legacy spec
flattens those. The previous transform only patched query parameters
on instance routes; request bodies drifted. This PR renames
`documentInstanceQueryParameters` → `matchLegacyOpenApi` and adds
schema normalization plus a parity test covering both.

## Design analysis

The new normalization at `public.ts:103-125` handles `anyOf`/`oneOf`
by stripping `null` branches, collapsing single-option unions, and —
notably — picking the `number` branch when a union mixes
`number | string` (the legacy `bigint` serialization shim). That last
heuristic is the only place where the transform makes a semantically
opinionated choice rather than a structural one; it deserves a code
comment explaining *why* number wins (presumably because legacy
clients posted JSON numbers and the string variant exists only for
bigint round-trip safety). Without that comment the next reader will
stare at `withoutNull.every((item) => item.type === "number" || item.type === "string")`
and not know whether it's safe to extend.

The `LegacyBodyRefParameters` denylist (`Auth`, `Config`, `Part`,
`WorktreeRemoveInput`, `WorktreeResetInput`) is a magic set with no
explanation. Each of those refs presumably already matches because
the legacy spec used the same `$ref` form, but a one-line comment
listing the contract — "these schemas are referenced by name in the
legacy spec, so do not inline them" — would prevent future
regressions when someone adds a new endpoint and wonders why their
new schema doesn't show up.

## Risks / suggestions

1. `structuredClone(spec.components.schemas[ref])` on every
   request body is fine for one-shot OpenAPI generation but worth
   noting if `matchLegacyOpenApi` ever runs in a hot path.
2. The new test `matches generated OpenAPI request body shape`
   compares serialized `requestBodyKey`s — that's a good
   round-trippable equality check, but consider also asserting
   that the diff list is empty *before* the transform, to prove
   the test would fail without the fix. Right now you could
   silently regress the transform and only the existing parameter
   test would catch it.
3. The PR title says "align" but the underlying motivation
   (presumably an SDK regen mismatch surfaced in CI) isn't in the
   description — worth a one-line "fixes #NNNN" reference if there
   is one.

## What I learned

When two API surfaces share a backend, the parity test must compare
*shape signatures*, not example payloads. The
`{required, content[type] → $ref|type|inline}` projection used here
is a nice pattern: it's tolerant to harmless variations (description
strings, ordering) but catches every structural drift that would
break a generated client.
