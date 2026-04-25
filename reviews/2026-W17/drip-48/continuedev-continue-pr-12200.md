# continuedev/continue#12200 — Added new supported models, and removed deprecated models for Ask Sage

- **URL**: https://github.com/continuedev/continue/pull/12200
- **Author**: alex-mcgraw-askSage
- **Head SHA**: `39bac103d582ee0cefb6fb2cf50f538014ff38c0`
- **Verdict**: `merge-after-nits`

## Summary

Bumps the Ask Sage provider's default model from `gpt-4o` to `gpt-4.1`
in `core/llm/llms/Asksage.ts:63`, replaces the catalog of supported
gov-cloud models in `gui/src/pages/AddNewModel/configs/models.ts`
(GPT-4o → GPT-4.1, GPT-4o mini → GPT-4.1 mini, GPT-4o-gov → GPT-5.1
gov, GPT-3.5 → GPT-5.4, GPT-3.5 gov → GPT-5.4 nano), and updates the
docs at `docs/customize/model-providers/more/asksage.mdx` plus
`gui/src/pages/AddNewModel/configs/providers.ts`. Bulk of the diff
(~3000 lines) is the four `package-lock.json` files that got swept
along.

## Reviewable points

- `core/llm/llms/Asksage.ts:63`: the `defaultOptions.model` flip from
  `"gpt-4o"` to `"gpt-4.1"` will silently change behaviour for any
  existing user who has the Ask Sage provider configured but did not
  pin a model. That's the intended fix (gov-cloud no longer hosts
  `gpt-4o`) but a release-notes entry is warranted.
- `gui/src/pages/AddNewModel/configs/models.ts`: each replaced model
  entry retains the same array index but changes both the catalog key
  and the user-visible `title`. The `contextLength`/`maxTokens` jumps
  (e.g. GPT-3.5 from whatever it was → 400000/200000 for GPT-5.4) are
  the right shape for the new family, but downstream cost/usage
  estimators that key off these values should be re-validated.
- The `package-lock.json` churn across `docs/`, `extensions/vscode/`,
  `gui/`, and `packages/openai-adapters/` looks unrelated to the model
  catalog change. If these were picked up from a stale base branch
  they should be dropped to keep the diff reviewable; if they were
  intentionally regenerated they deserve a separate commit.
- The MDX file shows `gpt4-gov` → `gpt-4.1-gov` consistently. Worth
  spot-checking that Ask Sage's API actually accepts the hyphenated
  form `gpt-4.1-gov` rather than the prior `gpt4-gov` slug.
- No corresponding test file change. The model catalog has no unit
  tests in the visible diff; that's likely consistent with how the
  existing entries are managed but means the only signal that the
  catalog stayed self-consistent is human review.

## Rationale

This is a vendor-driven catalog refresh and the contributor is from
`alex-mcgraw-askSage`, so the model-name authority is correct. The
default-model flip and the bundled lockfile changes are the two items
to address before merge: extract the lockfile updates and call out the
default-model change in the release notes / PR description.
