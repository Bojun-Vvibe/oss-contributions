---
pr: https://github.com/QwenLM/qwen-code/pull/3645
sha: cb0629a9
diff: +506/-12
state: OPEN
---

## Summary

Reorders OpenAI auth's model precedence from `argv.model > OPENAI_MODEL > QWEN_MODEL > settings.model.name` to `argv.model > settings.model.name > OPENAI_MODEL > QWEN_MODEL`, so that an `OPENAI_MODEL` environment variable no longer silently overrides a model the user picked via `/model` (which writes to `settings.model.name`). Adds 494 lines of behavioral pin tests covering eight scenarios (Cases A–H) plus four edge cases.

## Specific observations

- `packages/cli/src/utils/modelConfigUtils.ts:84` — docstring updated to match the new order. Good hygiene; the docstring is the contract surface most users actually read.
- `packages/cli/src/utils/modelConfigUtils.ts:95-122` — the implementation refactor is the right shape: instead of the old "pass `env` to `resolveModelConfig` and hope its precedence matches ours", the new code resolves the target model **first** with explicit `if/else if` precedence, then computes a `filteredEnv` that strips `OPENAI_MODEL`/`QWEN_MODEL` whenever the resolved model didn't actually come from those env vars. That filter at `:113-120` is the load-bearing line — without it, `resolveModelConfig` would re-apply env precedence internally and undo the fix:
  ```ts
  if (
    resolvedModel !== env['OPENAI_MODEL'] &&
    resolvedModel !== env['QWEN_MODEL']
  ) {
    delete filteredEnv['OPENAI_MODEL'];
    delete filteredEnv['QWEN_MODEL'];
  }
  ```
  Subtle but correct: the env vars are only forwarded when they actually were the source of truth.
- The refactor changes when `modelProvider` lookup happens — old code looked it up from `argv.model || settings.model?.name` and ignored env entirely; new code looks it up from `resolvedModel` which can now be the env value when neither argv nor settings has a model. That's a behavior change beyond the stated precedence fix: previously an env-set model would have a `modelProvider: undefined` in `configSources`; now it gets a real provider lookup. This is almost certainly the intended behavior (env-selected model should also resolve `generationConfig`/`envKey` overrides) but it should be called out in the PR description as a bonus fix, since downstream code paths that were tolerating `modelProvider: undefined` may now hit a populated provider with surprising `generationConfig`. The new Case-G/H tests should cover this if they assert on `generationConfig`-derived values.
- `modelConfigUtils.test.ts +494/-12` — the test density is excellent (8 cases + 4 edge cases for one precedence change). Each case has its own `it()` block with a header comment naming the scenario, and the assertions use `expect.objectContaining({ modelProvider: ... })` rather than full-shape matches, which is the right granularity for a precedence test (low brittleness, high specificity on the contract under test). Spot-checking Case A asserts `settings.model.name` wins over `OPENAI_MODEL`; Case B asserts env wins when settings is unset; Edge Case 1 asserts argv wins over both. That's the precedence triangle pinned correctly.
- `packages/cli/src/utils/modelConfigUtils.test.ts:493` — `// trigger rebuild` trailing comment is dev-loop noise that should not land. Strip before merge.
- The line `model: undefined as unknown as Settings['model']` (used to coerce undefined into the typed slot at `:534, :593, :675, ...`) appears repeatedly across the new tests. This is a legitimate test-only escape hatch but suggests `Settings['model']` should be `name?: string` (already optional via `model?:`) rather than requiring a sentinel cast — a follow-up to widen the test helper's `makeMockSettings` to accept `Partial<Settings>` recursively would clean this up.

## Verdict

`merge-after-nits` — the precedence fix is correct, the env-filter trick is exactly the right way to make `resolveModelConfig` honor the new order without re-implementing it, and the test coverage is genuinely thorough. Two small things before merge: (1) drop the `// trigger rebuild` comment at line 493, (2) call out the `modelProvider` lookup behavior change (env-resolved model now gets a provider lookup) in the PR description so the next bisector doesn't get confused.

## What I learned

When a downstream resolver has its own precedence logic, the cleanest way to override it from upstream is to *strip* the inputs the downstream logic uses, rather than try to outrank it positionally. This PR's `delete filteredEnv['OPENAI_MODEL']` pattern is more robust than re-ordering arguments because it makes the intent explicit at the boundary: "these env vars did not contribute to the decision, so the resolver must not see them."
