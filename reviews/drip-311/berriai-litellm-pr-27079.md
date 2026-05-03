# BerriAI/litellm #27079 — fix(gemini): generate unique tool_call_ids in GoogleGenAI adapter

- PR: https://github.com/BerriAI/litellm/pull/27079
- Author: netbrah
- Head SHA: `9cf922b096883a2db95be0770379edc694b8713b`
- Updated: 2026-05-03T09:08:36Z

## Summary
Fixes a correctness bug in `litellm/google_genai/adapters/transformation.py:_transform_contents_to_messages` where every functionCall was assigned `tool_call_id = f"call_{func_call.get('name', 'unknown')}"`, and every functionResponse was matched with `tool_call_id = f"call_{func_response.get('name', 'unknown')}"`. This collapses two parallel tool calls of the same name into one id, so the second response cannot be correctly matched. New behavior: model-role functionCall parts get unique IDs (`call_{uuid.uuid4().hex[:24]}`) tracked in a per-name FIFO queue (`pending_tool_call_ids: Dict[str, List[str]]`), and user-role functionResponse parts pop matching IDs from that queue in order. Refactors the giant for-loop into `_transform_user_parts` and `_transform_model_parts` helpers (good readability win).

## Observations
- `litellm/google_genai/adapters/transformation.py:384-388`: `pending_tool_call_ids: Dict[str, List[str]] = {}` — keyed by function name with a FIFO queue of pending IDs. **Correctness assumption:** the Gemini contents array preserves the order in which parallel functionCalls were emitted, and the matching functionResponses also arrive in the same order. This is the documented Gemini behavior, so the FIFO match is correct. Worth a one-line comment citing the Gemini docs to lock in the assumption.
- `litellm/google_genai/adapters/transformation.py:438-447`: the matching logic `pending_ids = pending_tool_call_ids.get(func_name, []); if pending_ids: matched_id = pending_ids.pop(0) else: matched_id = f"call_{uuid.uuid4().hex[:24]}"` — the fallback branch handles "functionResponse with no preceding functionCall" by minting a fresh id. Reasonable but worth a `verbose_logger.warning` because it indicates malformed input from upstream.
- The split into `_transform_user_parts(parts, messages, pending_tool_call_ids)` and `_transform_model_parts(parts, messages, pending_tool_call_ids)` is a real readability win — the original method was 120+ lines of nested branching. The shared `pending_tool_call_ids` dict is passed by reference, which is the right shape for cross-turn matching.
- `_transform_model_parts` (referenced but not fully shown in diff truncation) presumably populates `pending_tool_call_ids[func_name].append(matched_id)` when emitting a `tool_call`. Confirm that path exists and that the same `uuid.uuid4().hex[:24]` style is used to keep id format consistent.
- Truncation trims my view of the model-side population logic and the new tests. **Reviewer must confirm:** (1) there are tests for the parallel-same-name case (two `getWeather` calls in one turn → two distinct ids → two responses correctly matched); (2) there are tests for the fallback orphan-response case; (3) IDs match the expected `call_*` shape so downstream OpenAI-shape consumers don't break.
- Potential concern: any caller that *depended* on the old deterministic `call_{name}` id format (e.g. for caching or replay) breaks here. Migration note in the PR description would help. Realistically nobody should depend on it because it was buggy, but flag it.
- `import uuid` at line 2 — clean addition, no other dep changes.

## Verdict
`merge-after-nits`
