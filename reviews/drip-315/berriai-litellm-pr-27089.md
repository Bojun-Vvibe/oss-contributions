# BerriAI/litellm PR #27089 — feat(anthropic): enable native JSON mode for Claude 4.7 and sync max_completion_tokens

- Link: https://github.com/BerriAI/litellm/pull/27089
- Head SHA: `8f60e503c1363f9d67f5719bd8634e3dd0caec22`
- Author: koushik1124
- Size: +78 / −0 (2 files)

## Files changed
- `litellm/llms/anthropic/chat/transformation.py` — `+7 / −0` (extend the model-name allowlist for native JSON mode)
- `tests/test_litellm/test_parity_gap.py` — new file (71 lines, 3 tests)

## Reasoning

The transformation diff is a strict additive widening of an existing model-name set:

`litellm/llms/anthropic/chat/transformation.py:1057` adds variants:
```
"sonnet-4.7",
"sonnet-4-7",
"sonnet_4.7",
"sonnet_4_7",
"4.7-sonnet",
"4-7-sonnet",
"4_7_sonnet",
```

This is consistent with the existing pattern just above (`sonnet-4-6`, `sonnet_4.6`, `sonnet_4_6`). The check is presumably `any(... in model)` substring-style — that's also fragile but it's the established convention in this file, so this PR is just keeping parity.

Concerns:

1. **No `opus-4.7` variants despite the PR description claiming "Claude 4.7 variants" plural.** The PR description says: "Claude 4.7 variants were missing... `sonnet-4.7`, `opus-4.7`, etc." but only sonnet variants are added in the diff. If Anthropic ships a 4.7-opus, the same allowlist gap will reproduce. Either add `opus-*-4.7` variants now or scope the PR title/description to "Sonnet 4.7" only.

2. **Test at `tests/test_litellm/test_parity_gap.py:5` uses a fabricated date string `"claude-4-7-sonnet-20261031"`.** This is fine for the test (it just needs to match the substring check) but it's worth flagging that no real model id is being exercised — if Anthropic's actual id format diverges (e.g. uses a different separator), the substring check may still miss real production strings.

3. **The Bedrock and Vertex parity tests (`test_bedrock_max_completion_tokens_parity`, missing test for Vertex)** in the new file don't actually exercise the Anthropic transformation under review — they assert `AmazonConverseConfig` and `VertexAIConfig` separately. That's fine as additional coverage but they have no causal link to the diff in this PR. The PR description claims Vertex parity was "verified" but I see no `test_vertex_max_completion_tokens_parity` in the diff. Would prefer either adding the missing Vertex test or scoping the PR title to "Anthropic + Bedrock parity audit".

4. **Style nit: `tests/test_litellm/test_parity_gap.py:1` opens with a blank line and does not have a module docstring.** Minor.

## Verdict

`merge-after-nits`

Useful additive change and the test pattern is sound. Before merging, please:
- Add `opus-4.7` variants to the allowlist or narrow the PR title to "Sonnet 4.7".
- Either add the claimed Vertex test or remove "Vertex AI/Gemini" from the PR description.
- Remove the misleading Bedrock test if the Bedrock allowlist isn't actually changed in this PR (I don't see a Bedrock transformation diff here).
