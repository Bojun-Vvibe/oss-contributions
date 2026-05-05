# block/goose#9027 — fix(ci): mark openai/gpt-5 smoke test as flaky

- **URL:** https://github.com/block/goose/pull/9027
- **Head SHA:** `185c6187cfd1fbf371c46e9fd169ee530968f1ae`
- **Files touched:** 1 (+6 / −1)
  - `ui/desktop/tests/integration/test_providers_lib.ts`

## Summary

The `Smoke Tests (Code Execution)` job has been failing repeatedly on `main` and on PR branches once they merge `main`. The failure is consistently in the `openai / gpt-5` test case: the model spends its turn calling `list_functions` and `get_function_details` for `Memory` and `Todo`, and never reaches `code_execution` before the 55s `runGoose` timeout fires. This PR marks that single model entry as `{ flaky: true }`, mirroring the established pattern from #8837 already used for `gpt-3.5-turbo`, `qwen/qwen3-coder:exacto`, `gemini-2.5-flash`, `gemini-3-pro-preview`, and `nvidia/nemotron-3-nano-30b-a3b:free`.

## Line-level observations

- `test_providers_lib.ts:69-77` — the inner array under `provider: 'openai'` is reformatted from a single line to multi-line, with `'gpt-5'` rewritten as `{ name: 'gpt-5', flaky: true }`. The string-vs-object polymorphism is already supported by the existing reader (the entry for `'gpt-3.5-turbo'` was already an object, and the other listed models are still bare strings). Reading the surrounding diff context, the model-list shape accepts either form, so no consumer change is needed.
- The `flaky: true` flag's effect (per the PR description and the cited #8837 prior art): vitest test timeout is bumped to 90s so the internal 55s `runGoose` rejection fires first, and the rejection is caught and logged as ``console.warn(`Flaky test ... failed (allowed): ${err}`)`` instead of failing the build. The fast non-code-exec smoke test (which gpt-5 passes in ~5s) is unaffected — the flag only changes timeout behavior, not correctness assertions.
- Six failing run links are cited in the PR body (latest plus five prior runs from `main`) and the PR notes that all 14 other non-flaky models pass code-exec consistently. The evidence that the issue is gpt-5-specific (model spending its turn introspecting tools rather than executing) is solid.

## Concerns

- This is a CI-stability bandage, not a fix for the underlying behavior (gpt-5 introspects tools instead of calling code_execution under the goose harness). Worth filing a follow-up issue against the prompt/system-message that's nudging gpt-5 toward `list_functions` first, or against the model selection if gpt-5 isn't actually expected to be the right code-exec target. The PR description does not mention such a follow-up.
- The flaky-allowlist now contains 6 models. At some threshold, "flaky" stops being a useful signal and starts hiding real regressions. A maintainer comment establishing the policy (e.g. "we accept any model going flaky if it doesn't fail the same way 3 runs in a row") would help frame future additions.
- The test plan checkboxes in the PR are unchecked. Author should confirm at least the regular Smoke Tests still pass for gpt-5 (which the PR claims succeeds in ~5s) before maintainer merge.

## Verdict

**merge-as-is**

## Rationale

Six-line CI fix that mirrors an established, well-understood pattern (#8837) for the same model-list. Evidence is concrete (six run links, tool-introspection root-cause identification, ~5s vs >55s asymmetry between the two smoke jobs). Build is currently red on `main`; this unblocks PR authors immediately. The follow-up concerns about underlying model behavior and flaky-allowlist policy are real but should be tracked separately, not used to block this stop-the-bleeding change.
