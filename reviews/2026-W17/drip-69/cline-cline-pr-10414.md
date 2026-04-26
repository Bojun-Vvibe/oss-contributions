# cline/cline #10414 — Update: Deepseek reasoner check to include v4-pro

- **Author:** richpix
- **Head SHA:** `da3bb4f76a38ef096f25cf27c04497746aac6f9c`
- **Size:** +2 / -2 (1 file)
- **URL:** https://github.com/cline/cline/pull/10414

## Summary

Two-line change to `DeepSeekHandler` so that the new `deepseek-v4-pro`
model receives the same special treatment as the legacy
`deepseek-reasoner`: messages get formatted with `addReasoningContent`,
and the `temperature: 0` parameter is omitted (reasoning models don't
honour it). The PR body also describes config-file changes adding
`deepseek-v4-flash`/`deepseek-v4-pro` and flipping the default — but
those changes are *not in this diff*, only the handler tweak is.

## Specific findings

- **The intent is correct.**
  `src/core/api/providers/deepseek.ts:84` — promoting the check from
  `model.id.includes("deepseek-reasoner")` to also include
  `"deepseek-v4-pro"` is the right gate for both the message-formatting
  branch and the temperature-omission branch. The matching change at
  line 98 — switching the conditional from
  `model.id === "deepseek-reasoner"` to the new `isDeepseekReasoner`
  variable — also fixes a minor pre-existing inconsistency where the
  message-formatting branch used `includes` but the temperature branch
  used `===`. Good cleanup.

- **There is a real syntax bug in the diff.** Look at the +/- around
  line 98:

  ```
  -    ...(model.id === "deepseek-reasoner" ? {} : { temperature: 0 }),
  +    ...(isDeepseekReasoner ? {} : { temperature: 0 })
       ...getOpenAIToolParams(tools),
  ```

  The trailing comma after `{ temperature: 0 })` was removed but the
  next line `...getOpenAIToolParams(tools),` is still a sibling spread
  in the same object literal. Without the comma, this is `{ temperature:
  0 } ...getOpenAIToolParams(tools)` — a TypeScript parse error. CI
  should catch it on `tsc --noEmit`; the author should restore the comma
  before review proceeds. **This is the headline blocker.**

- **PR description claims more than the diff does.** The body advertises
  model-definition additions (`deepseek-v4-flash`, `deepseek-v4-pro`
  with 1M context, 384K max output, promotional 75% discount pricing
  through 2026-05-05) and a default-model flip. The diff only contains
  the handler change — there's no `deepseekModels` table edit, no
  default flip. Either the PR is missing commits or the description is
  describing a parallel/sibling PR. Reviewers should ask the author to
  reconcile before merging, otherwise the actual merged behaviour will
  be "v4-pro gets reasoner treatment but is unselectable" because no
  model entry references it.

- **Backward-compat claim is fine.** Existing `deepseek-chat` /
  `deepseek-reasoner` ID matching is preserved, so the legacy path is
  intact. The handler change is purely additive in semantic effect once
  the syntax bug is fixed.

## Verdict

`request-changes`

(Restore the missing comma at line 98 — the diff as posted does not
compile. Also reconcile the description against the actual diff: either
push the missing model-table commits or trim the description so future
readers don't expect bundled model definitions that aren't there.)
