# PR #26069 — fix(core): handle non-string model flags in resolution

- **Repo**: google-gemini/gemini-cli
- **PR**: #26069
- **Head SHA**: `c29089fa`
- **Author**: Adib234
- **Size**: +116 / -2 across 4 files
- **URL**: https://github.com/google-gemini/gemini-cli/pull/26069
- **Verdict**: **merge-after-nits**

## Summary

When `--model` is passed twice on the CLI, yargs returns it as
`string[]`, not `string`, and the previous resolution path passed
that array straight into `resolveModel(...)` and downstream
`isCustomModel(...)` checks. The downstream callers blew up with
"`.includes` is not a function on string" or silently coerced via
`String(arr)` (yielding e.g. `"gemini-1.5-pro,gemini-2.0-flash"`,
which then matched no known model and routed to the custom-model
fallback for the wrong identifier). Fix is two layers: at the CLI
boundary (`config.ts:818-826`) flatten an array `argv.model` to
its last element via `at(-1)`, and inside `resolveModel`
(`models.ts:109-119`) repeat the defensive flatten so non-CLI
callers (programmatic API, settings file with a misshapen value)
get the same coercion.

## Specific changes

- `packages/cli/src/config/config.ts:818-826` — replaces direct
  `const specifiedModel = argv.model || ... || ...` with a two-step
  `rawModel` then `specifiedModel = Array.isArray(rawModel) ?
  String(rawModel.at(-1) ?? '').trim() || '' : rawModel === undefined
  ? undefined : (String(rawModel ?? '').trim() || '')`. The
  `undefined → undefined` carve-out is correct: callers downstream
  treat `undefined` as "fall through to default" and `''` as
  "explicitly empty, error". Conflating those would regress the
  no-flag-passed path.
- `packages/core/src/config/models.ts:109-119` — the same flatten
  applied inside `resolveModel`. Identical semantics:
  array-of-strings → last; non-string scalar → `String(value).trim()
  || ''`; string → trim. The `experimental.dynamicModelConfiguration`
  branch uses `normalizedModel` so the experimental path can't
  bypass the coercion.
- `packages/cli/src/config/config.test.ts:780-870` — two new tests
  in a `'Model resolution'` describe block:
  - `should handle multiple --model flags by taking the last one` —
    constructs a `CliArgs` with `model: ['gemini-1.5-pro',
    'gemini-2.0-flash'] as any`, calls `loadCliConfig`, asserts
    `config.getModel() === 'gemini-2.0-flash'`. Last-wins is the
    standard yargs/argparse semantics for repeat flags so this
    matches user expectation.
  - `should handle non-string model flags by coercing to string` —
    passes `model: true as any`, asserts `getModel() === 'true'`.
    This pins the coercion shape but the assertion is a little odd
    in that `true` arriving here would itself be a yargs misuse
    upstream; the better assertion would arguably be that an error
    is raised, but accepting and stringifying matches the existing
    `String(...)` cast behavior elsewhere in the file so the
    consistency argument wins.
- `packages/core/src/config/models.test.ts:273-449` — `isCustomModel`
  gains an "array doesn't throw" assertion plus a "last element
  classifies" assertion, and `resolveModel` gains a three-case
  parametrized test covering array, boolean, null. Good coverage.

## Risks

The defensive flatten at the `resolveModel` layer is a small but
real behavior change for any programmatic caller that *was*
relying on `resolveModel(['a','b'])` throwing — there isn't likely
to be any such caller because the previous behavior was
inconsistent (sometimes throw, sometimes silently mis-resolve)
but worth a CHANGELOG note.

Two diff-level nits before merge:

1. The boolean-coercion case (`model: true as any` → `'true'`) is
   defensively correct but probably never happens in practice (yargs
   never produces a bare boolean for a flag with a value), and the
   test pinning it elevates it to spec. Either delete the boolean
   test (and let `String(true)` collapse to `'true'` as an
   incidental behavior) or replace it with a `null`/`undefined`
   carve-out that's actually reachable — `model: null as any`
   currently produces `''` per the `String(null ?? '')` branch and
   that's the more interesting case to pin.

2. The two coercion call sites (`config.ts:818-826` and
   `models.ts:109-119`) are textually different
   (`String(rawModel.at(-1) ?? '').trim()` vs
   `String(requestedModel.at(-1) ?? '').trim()`). They should share
   a tiny `coerceToModelString(value: unknown): string` helper in
   `models.ts` so a future tweak to the coercion semantics doesn't
   need to be applied in two places (and risk drift).

## Verdict rationale

Real bug, real fix, sensible test coverage, low blast radius. The
two nits are about consolidation and test selection, not
correctness — neither blocks merge. Land it after the helper
extraction or with a follow-up issue tracked.
