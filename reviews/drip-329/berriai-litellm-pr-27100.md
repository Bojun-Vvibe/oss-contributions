# BerriAI/litellm #27100 — fix(core_helpers): map vLLM 'repetition' finish_reason to 'stop'

- SHA: `1cd2f5cc5148326c88a90e2e901a110eddd143cd`
- State: OPEN, +39,332/-1,608 (the **substantive** change is +2/-0 in `core_helpers.py` plus +9/-0 in its test file; the rest is a noisy rebase carrying many unrelated `litellm/proxy/*`, `litellm/llms/vertex_ai/*`, `litellm/llms/anthropic/*`, and `_lazy_openapi_snapshot.json` changes from main)
- Substantive files: `litellm/litellm_core_utils/core_helpers.py`, `tests/test_litellm/litellm_core_utils/test_core_helpers.py`

## Summary

vLLM's `RepetitionDetectionParams` server-side stop emits `finish_reason: "repetition"`, which is missing from `_FINISH_REASON_MAP`. Result: every affected row logs `WARNING - core_helpers.py:108 - Unmapped finish_reason 'repetition', defaulting to 'stop'`. PR makes the existing fallback explicit by registering the mapping, killing the warning flood.

## Notes

- `litellm/litellm_core_utils/core_helpers.py:182-183` — adds `"repetition": "stop",` with a comment `# vLLM (server-side RepetitionDetectionParams cuts off contiguous loops)`. Sits between the Bedrock `guardrail_intervened` mapping and the OpenAI passthrough block. Placement matches file convention of grouping by provider.
- The chosen target `"stop"` is correct relative to the file's existing convention. `length` is reserved for token-capacity reasons (`max_tokens`, `compaction`, `MAX_TOKENS`). Other server-side cut-offs (`network_error`, `MALFORMED_RESPONSE`, `TOO_MANY_TOOL_CALLS`, `ERROR`) all already route to `stop`. Repetition-loop termination is a server-decided cut-off, not a token-capacity event, so `stop` is semantically right.
- `tests/test_litellm/litellm_core_utils/test_core_helpers.py:38106-38112` — adds `class TestMapFinishReasonVLLM` with `test_repetition` asserting `map_finish_reason("repetition") == "stop"`. New test class (not appended to an existing one), which is the pattern used by sibling classes in the same file (`TestMapFinishReasonOpenAIPassthrough` at line 38115). Consistent.
- Test docstring explicitly references the `RepetitionDetectionParams` cause and the warning message it suppresses. Future debuggers will grep for `Unmapped finish_reason 'repetition'` and find the test. Good.
- **Rebase noise concern:** the diff includes massive unrelated changes (`litellm/proxy/_lazy_openapi_snapshot.json` +31,651, `litellm/proxy/proxy_server.py` +353/-229, `litellm/proxy/management_endpoints/ui_sso.py` +296/-53, `litellm/proxy/litellm_pre_call_utils.py` +170/-12, `tests/test_litellm/proxy/_experimental/mcp_server/test_db_credentials.py` +402, ...). These look like main-branch commits that landed since the branch was forked. Reviewer must request a clean rebase or a squash-merge; otherwise this PR's blame and revert story is broken. The PR body honestly describes only the 2-line change, so the noise is unintentional.
- No semantic conflict with the unrelated changes — none of them touch `_FINISH_REASON_MAP` or `map_finish_reason`. So a rebase will Just Work; this is purely a hygiene issue.
- No backwards-compat concern: callers that today see the warning + `stop` will tomorrow see no warning + `stop`. Behavior is preserved bit-for-bit aside from log volume.
- No telemetry impact: `stop` is already what consumers receive via the file's `else: log warning, return "stop"` fallback path on master.

## Verdict

`merge-after-nits` — the actual fix is correct and minimal. Block on a clean rebase / interactive squash so the merged commit is +2/-0 (plus +9/-0 test), not +39k/-1.6k. Without that, bisects and reverts will be a nightmare.
