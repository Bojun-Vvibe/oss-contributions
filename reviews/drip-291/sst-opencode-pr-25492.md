# sst/opencode PR #25492 — fix(provider): limit JSON schema depth for Moonshot/Kimi

- URL: https://github.com/anomalyco/opencode/pull/25492
- Head SHA: `2e1517dfccf8adf64c0b544c97f76469b3e01449`
- Author: nateGeorge
- Repo: sst/opencode

## Summary

Adds depth-limiting to the existing Moonshot/Kimi schema sanitizer so tool
schemas that exceed Kimi K2.6's ~10-level nesting cap get flattened into a
loose `object`/`array` shape instead of being sent verbatim and rejected.

## Specific notes

- `packages/opencode/src/provider/transform.ts:1104` defines
  `MAX_DEPTH = 9` with a comment noting Kimi rejects > 10 and one level of
  headroom — sensible. Worth confirming the floor empirically; the PR body
  should cite the failing payload that motivated the cutoff.
- `transform.ts:1106-1117` `isComplex` checks for `properties / anyOf /
  oneOf / allOf / items / $defs / definitions` — covers the standard JSON
  Schema container kinds. Note that `prefixItems` (2020-12) is **not**
  included; if Kimi ever sees a prefixItems array nested deeply this would
  not flatten it. Minor.
- `transform.ts:1119-1124` `flatten` collapses arrays to
  `{ type: "array", items: { type: "object", additionalProperties: true } }`
  and everything else to `{ type: "object", additionalProperties: true }`.
  This is information-destroying but acceptable as a fallback; would be
  worth logging at warn-level once per request so users notice that their
  tool's deep types are being lost.
- Recursion is now `depth + 1` for both the array branch (line 1130) and
  the object branch (line 1140) — consistent. Note that the `$ref` early
  return at line 1133 does **not** advance depth, which is correct since
  Moonshot expands refs server-side.
- Tests at `transform.test.ts:1000-1129` cover the deep-nesting case and a
  realistic anyOf-with-object cascade. The first test asserts depth
  `>= 4 && < 12`, which is loose; could tighten to
  `expect(depthReached).toBe(MAX_DEPTH + 1)` to catch off-by-one regressions.

verdict: merge-after-nits
