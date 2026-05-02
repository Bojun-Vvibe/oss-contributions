# Review: BerriAI/litellm #27047 — fix(langfuse_otel): handle pydantic Message in set_messages

- Repo: BerriAI/litellm
- PR: #27047
- Head SHA: `f5dc4407538e87b6e91a125092f6fdeee90e2344`
- Author: weiguangli-io
- Size: +100 / -1 across 2 files

## What it does
Replaces `json.dumps(input)` in `LangfuseLLMObsOTELAttributes.set_messages`
with `safe_dumps`, which knows how to serialize pydantic `Message` objects
(and other non-JSON-native types). Adds three regression tests (issue #26977).

## File-level notes

**`litellm/integrations/langfuse/langfuse_otel_attributes.py` @ L17, L100**
```py
+from litellm.litellm_core_utils.safe_json_dumps import safe_dumps
...
- safe_set_attribute(span, "langfuse.observation.input", json.dumps(input))
+ safe_set_attribute(span, "langfuse.observation.input", safe_dumps(input))
```
- Minimal, targeted fix. `safe_dumps` is the established codebase convention
  for serializing objects that may contain pydantic models.
- The unused `import json` at the top of the file should also be cleaned up
  if it's no longer referenced after this change — confirm nothing else in
  the module uses it.

**`tests/test_litellm/integrations/langfuse/test_langfuse_otel_attributes.py` (new file, +98)**
- Three tests cover: pydantic `Message` only, plain-dict messages
  (regression guard), and pydantic `Message` + tools combined. Asserts the
  serialized payload is parseable JSON and that key fields survive — good.
- `sys.path.insert(0, os.path.abspath("../../../.."))` is fragile; if test
  collection happens from a different cwd this breaks. Most other litellm
  tests rely on the conftest `sys.path` setup. Recommend deleting the
  insert line.
- `MagicMock` for `span` works but `span.set_attribute.assert_called_once()`
  + reading `call_args[0]` is fine; consider also asserting the JSON
  payload's `messages[0].content` matches to lock the contract end-to-end
  (already done in test 1, missing in tests 2 and 3 for content field).

## Risks
- Behavior change: payloads that previously raised `TypeError` now succeed.
  No path that previously succeeded changes shape — `safe_dumps` falls back
  to `json.dumps` semantics for plain types.

## Verdict: `merge-after-nits`
Drop the manual `sys.path.insert`, remove unused `json` import if any, and
this is good to land.
