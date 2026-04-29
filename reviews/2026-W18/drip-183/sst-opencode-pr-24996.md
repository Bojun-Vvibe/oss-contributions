---
pr: sst/opencode#24996
sha: e928fe64cd89c541dca8e166b84516579dbc1ca0
verdict: merge-after-nits
reviewed_at: 2026-04-30T00:00:00Z
---

# feat: add Mistral Medium 3.5 with reasoning support

URL: https://github.com/sst/opencode/pull/24996
Files: `packages/opencode/src/provider/transform.ts`, `packages/opencode/test/provider/transform.test.ts`
Diff: 21+/3-

## Context

`ProviderTransform.variants` previously gated Mistral reasoning to two literal
substring checks for `mistral-small-2603` and `mistral-small-latest`
(`transform.ts:763`). Mistral has since shipped Medium 3.5 with the same
adjustable-reasoning capability, and the gate was silently dropping the
`high` reasoning-effort variant for that model.

## What's good

- The refactor from chained `&&` substring checks to a `MISTRAL_REASONING_IDS`
  array + `Array.some(includes)` (`transform.ts:763-764`) is the right shape:
  every future Mistral reasoning model is one entry, not a new boolean clause.
  Adding "Codestral 2509 reasoning" tomorrow is a one-line edit.
- The `capabilities.reasoning` short-circuit at `transform.ts:761` is
  preserved, so `variants()` still returns `{}` for non-reasoning Mistral
  builds — no regression on the negative-case test at
  `transform.test.ts:2304+`.
- The new test `mistral-medium-3.5 with reasoning returns variants`
  (`transform.test.ts:2277-2293`) constructs a model with the exact
  `api.id = "mistral-medium-3.5"` shape and asserts the `{high: {reasoningEffort: "high"}}`
  payload — covers the fix end-to-end.

## Nits

- `transform.ts:763` keeps using `String.includes` rather than equality on
  `mistralId`. With Medium 3.5 in the list, a hypothetical model
  `mistral-medium-3.5-preview-mini-tool` would also pick up reasoning even
  if Mistral disables it for that variant. Tighten to either an
  endsWith/exact-match or anchor the strings with version-prefix
  understanding.
- The constant `MISTRAL_REASONING_IDS` is declared inside the variant
  closure (`transform.ts:763`); lift it to module-scope so the next
  reasoning gate (Codestral, Devstral) can compose with it instead of
  copy-pasting.
- The renamed test `mistral models with reasoning support return variants`
  (`transform.test.ts:2260`) is now plural but only exercises Small. Either
  parametrize across `MISTRAL_REASONING_IDS` to keep the name honest, or
  revert the rename.

## Verdict reasoning

Single-purpose model-list update with parametrized regression test, tight
blast radius, no risk to existing Mistral Small reasoning users. Substring
matching laxness is the only real follow-up worth a re-push, and only if
the maintainers care about the case where Mistral ships a non-reasoning
sub-variant under a name that contains `mistral-medium-3.5`.
