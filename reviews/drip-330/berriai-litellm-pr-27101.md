# BerriAI/litellm #27101 — fix(core_helpers): map vLLM 'repetition' finish_reason to 'stop'

- SHA: `9a18172d371f50603f69ed58ac636fd7259354f3`
- State: OPEN, +11/-0 across 2 files
- Replaces: #27100

## Summary

Two-line fix: registers `"repetition": "stop"` in the `_FINISH_REASON_MAP` so vLLM's `RepetitionDetectionParams` cutoff stops emitting `Unmapped finish_reason 'repetition', defaulting to 'stop'` warnings on every affected row. Adds a one-line unit test asserting the mapping. Cleanly cut from `litellm_oss_staging` to drop the unrelated commits that contaminated the prior PR (#27100).

## Notes

- `litellm/litellm_core_utils/core_helpers.py:97-99` — adds `"repetition": "stop"` between the Bedrock `guardrail_intervened` entry and the `# OpenAI passthrough` block. Comment explains the provenance ("vLLM (server-side RepetitionDetectionParams cuts off contiguous loops)"). Placement is logical; alphabetical ordering isn't enforced elsewhere in this map.
- Choice of `"stop"` vs `"length"` is defensible and consistent with the file's existing convention: `length` is reserved for token-capacity reasons (`max_tokens`, `compaction`, `MAX_TOKENS`), while server-side cut-offs and error states (`network_error`, `MALFORMED_RESPONSE`, `TOO_MANY_TOOL_CALLS`, `ERROR`) all route to `stop`. Repetition detection is closer to the latter category. PR body documents this rationale clearly.
- Arguable counter-position: a downstream cost-tracker or eval harness might want to *distinguish* repetition-stopped completions from natural stops (they're often lower quality and you may want to discard them). Mapping to `stop` makes them invisible. Worth a one-line note in the PR or a follow-up issue acknowledging that consumers needing the distinction should look at the raw `provider_finish_reason` field (if exposed) rather than the mapped one.
- `tests/test_litellm/litellm_core_utils/test_core_helpers.py:146-152` — new `TestMapFinishReasonVLLM.test_repetition` is a single assertion. Adequate for a one-line map entry. Consistent with the surrounding `TestMapFinishReasonOpenAIPassthrough` style (which uses `parametrize`); for one value parametrize is overkill.
- The PR mentions this replaces #27100 because that branch retargeted base and accumulated unrelated commits. The commit history here looks clean (single fix commit cut from `litellm_oss_staging`), so the maintainer can fast-merge without worrying about cherry-pick artifacts.
- No test for the negative case (the warning *not* firing) — the map test alone doesn't prove the warning is gone. A `caplog` assertion that `map_finish_reason("repetition")` emits zero warnings would lock the actual user-visible behavior. Optional.
- Nit: PR body says "register `\"repetition\": \"stop\"`" but the comment uses "RepetitionDetectionParams". Either capitalization is fine; the comment is the canonical name from vLLM source.

## Verdict

`merge-as-is` — surgical two-line fix with explicit rationale, clean replacement of a contaminated prior PR, aligned with existing conventions in the same file, and a passing test. Optional follow-ups (caplog assertion, downstream-distinguishability note) are nice-to-haves and shouldn't block.
