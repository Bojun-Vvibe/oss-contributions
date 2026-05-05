# sst/opencode #25775 — fix(provider): keep tool-call and tool-result paired in anthropic normalize

- URL: https://github.com/sst/opencode/pull/25775
- Head SHA: `1a68eadc39772e6e51ecf3243c0c668003d98c7d`
- Author: codeg-dev
- Size: +82 / -4 across 2 files

## Comments

1. `packages/opencode/src/provider/transform.ts:140` — Renaming the conceptual unit from "tool-call" to "tool-call OR tool-result" is the right call. Anthropic's API requires `tool_use` and the matching `tool_result` to live together; the previous splitter would happily put the `tool_use` into a trailing assistant message while leaving the `tool_result` block stranded in the original message, which produces a 400 from Anthropic. Good catch.
2. `packages/opencode/src/provider/transform.ts:142` — The "any non-tool-shaped part exists from `first` onwards?" guard is now correct and symmetric with the partition predicate two lines below. Worth a one-line comment above this block explaining *why* tool-call and tool-result must travel together (link to the Anthropic API docs section on `tool_result` placement) — the diff itself reads as a stylistic refactor without that context.
3. `packages/opencode/src/provider/transform.ts:143-145` — The two `filter` calls are now mirror images. Consider extracting a single `isToolPart = (p) => p.type === "tool-call" || p.type === "tool-result"` local const at the top of the callback to avoid drift if a third tool-shaped part type ever lands (e.g. `tool-error`). Three references to the same disjunction in five lines is asking for a future bug.
4. `packages/opencode/test/provider/transform.test.ts:1500-1527` — Test 1 (paired call+result followed by text) is exactly the regression case. Good naming: "keeps tool-call and tool-result paired when splitting".
5. `packages/opencode/test/provider/transform.test.ts:1529-1556` — Test 2 (text interleaved between call and result) is the more interesting case — it documents that interleaved text gets pulled out into the leading message. Verify this matches what the model actually emits in practice; if Anthropic ever streams `text` *between* `tool_use` and `tool_result`, this normalization will reorder it relative to the tool boundary. Probably fine because the assistant doesn't speak between its own tool call and the tool's response, but worth a comment on the test.
6. `packages/opencode/test/provider/transform.test.ts:1558-1577` — Test 3 (no trailing text → no split) confirms the early-return path. Good negative test.
7. Missing: a test for the case where multiple `tool-call`/`tool-result` pairs are interleaved with text *before* the first pair. The current logic uses `findIndex` which only finds the *first* tool-shaped part; verify that two text-then-call-then-result-then-text-then-call-then-result sequences round-trip correctly. The current code probably puts the second text run into the wrong bucket.

## Verdict

`merge-after-nits`

## Reasoning

Correct fix targeting a real Anthropic API constraint, with three solid regression tests. The nits are quality-of-life: extract the predicate to avoid the three-way disjunction drift, add a doc comment pointing at the Anthropic spec, and add one more interleaving test for the multi-pair case. None of these block; the existing diff is strictly safer than `main`.
