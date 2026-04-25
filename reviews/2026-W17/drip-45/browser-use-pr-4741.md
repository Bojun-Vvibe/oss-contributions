# browser-use/browser-use #4741 — fix(anthropic-serializer): type tool_calls + raise on malformed data URL

- **PR:** https://github.com/browser-use/browser-use/pull/4741
- **Head SHA:** `b9548c9a956db4704a60221483eb8b186b7085bb`
- **Files changed:** 1 — `browser_use/llm/anthropic/serializer.py` (+10 / -2)

## Summary

Two small hardening changes in the Anthropic serializer: (a) explicit `ValueError` with a helpful message when a base64 data URL has no comma separator, instead of an opaque `unpack` error; (b) explicit `dict[str, object]` annotation on `input_obj` so the `json.JSONDecodeError` fallback (which produces `dict[str, str]`) widens correctly under dict-invariance, and import + use of the `ToolCall` type for the `tool_calls` parameter.

## Line-level call-outs

- `browser_use/llm/anthropic/serializer.py:21-24` — the new guard correctly surfaces the malformed-URL case. Good touch truncating the URL to 80 chars in the message — base64 payloads can be huge, and the prior crash logs were probably hostile. Minor: `url[:80]` will mid-cut a multibyte char; if URLs are guaranteed ASCII (they are, per RFC) this is fine, otherwise consider `url[:80].rsplit(...)`. Not blocking.
- `:21` — no tests added for the new branch. A one-line `pytest.raises(ValueError, match="missing comma")` would lock the contract.
- `:32` — annotating the parameter as `list[ToolCall]` is a strict improvement over the prior implicit `Any`. Verify upstream callers always pass `ToolCall` (not raw `dict`); a quick `rg "_serialize_tool_calls_to_content("` would confirm.
- `:38-43` — the explicit `input_obj: dict[str, object]` annotation is exactly the right fix for the variance issue described in the comment. The comment itself is excellent — keep it. One quibble: the `try` immediately reassigns `input_obj`, so the annotation is purely for the type checker on the `except` branch. A reviewer unfamiliar with mypy might think it's dead code; the comment mitigates this.
- Missing context: the diff doesn't show what the `except json.JSONDecodeError:` branch does — if it falls through to `{"_raw": tool_call.function.arguments}` or similar, the widening matters; if it `raise`s, the annotation is unnecessary. Worth a glance at the surrounding lines to confirm the comment matches reality.

## Verdict

**merge-as-is**

## Rationale

Two small, surgical hardening changes that improve error messages and type safety with zero behavioural risk. The lack of a test for the new `ValueError` branch is the only nit, and it's barely worth blocking on for a 10-line PR. Comments are well-written and explain the *why*, which is rare for type-only fixes.
