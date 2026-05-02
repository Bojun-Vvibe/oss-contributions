# BerriAI/litellm PR #26978 — fix(langfuse_otel): serialize pydantic Message objects in set_messages

- Head SHA: `51d940b497bc71e6c339701d4404dca7a13b3d88`
- URL: https://github.com/BerriAI/litellm/pull/26978
- Size: +73 / -1, 2 files
- Verdict: **merge-as-is**

## What changes

`LangfuseLLMObsOTELAttributes.set_messages` (langfuse_otel_attributes.py:88-103)
previously did `prompt = {"messages": kwargs.get("messages")}` and then
fed it through `json.dumps`. When a caller passed
`list[litellm.types.utils.Message]` (a Pydantic model — and a documented
public API surface), `json.dumps` raised
`TypeError: Object of type Message is not JSON serializable`, the
attribute setter aborted, and the OTEL span dropped the input payload.

The fix: normalize each entry — if it's a `BaseModel`, call
`model_dump()`, otherwise pass through. The new code at
langfuse_otel_attributes.py:96-99 does exactly that:

```py
raw_messages = kwargs.get("messages") or []
messages = [
    m.model_dump() if isinstance(m, BaseModel) else m for m in raw_messages
]
```

Two new tests
(test_langfuse_otel.py:118-176): one asserting pydantic
`Message(role="user", content="hello")` doesn't raise and round-trips
through `json.loads(captured["langfuse.observation.input"])`, and one
pinning the existing dict path so this PR can't accidentally break it.

## What's good

- The bug fingerprint is real and load-bearing: with langfuse OTEL
  enabled and pydantic message inputs, *every* call would log the
  spammy "Failed to set OpenInference span attributes" error. The
  comment block at langfuse_otel_attributes.py:89-95 documents the
  exact failure mode and the symptom users see in their logs — future
  archaeologists will thank you.
- `BaseModel` check (not `litellm.Message` specifically) means future
  pydantic message subclasses (e.g. provider-specific extensions) are
  also handled without further patching.
- `kwargs.get("messages") or []` defends against `None` (the previous
  code would have happily produced `{"messages": None}` which dumps
  fine but is meaningless).
- The "passes through plain dicts" test (test_langfuse_otel.py:154-176)
  explicitly asserts the dict-path `payload["messages"]` is identity-
  equal to the input. Good belt-and-braces.
- `safe_set_attribute` is patched per-test so the assertions read the
  actual serialized OTEL value, not just `mock_span.set_attribute`
  call counts. Stronger than typical mock-call-count assertions.

## Nits (minor)

- `BaseModel` is presumably already imported at the top of
  `langfuse_otel_attributes.py` (the diff doesn't show the import
  block); if not, this PR needs to add `from pydantic import BaseModel`.
  Worth a quick check before merge.
- `m.model_dump()` is pydantic v2 syntax. If litellm still supports
  pydantic v1 in any compatibility path, prefer `m.dict()` with the
  v2 fallback — though as of late 2024+ litellm has been v2-only, so
  this is likely moot.
- For very large message lists (long chat histories), the list
  comprehension materializes everything before `json.dumps`. Not a
  regression, just an observation: previously the dict path also
  materialized through `json.dumps`, so the memory profile is
  unchanged.

## Risk

Very low. Strict additive normalization on a code path that was
producing incorrect telemetry. The two-test coverage (pydantic + dict)
locks down both shapes. No behavior change for users not on langfuse
OTEL.
