# BerriAI/litellm#26978 — fix(langfuse_otel): serialize pydantic Message objects in set_messages

- **PR**: https://github.com/BerriAI/litellm/pull/26978
- **Author**: taxfree-python
- **Head SHA**: `10fd95faad625fc59173625944b213b1bb9b3bbd`
- **Files**: `litellm/integrations/langfuse/langfuse_otel_attributes.py` (+13 / -1), `tests/test_litellm/integrations/test_langfuse_otel.py` (+60 / -0)
- **Verdict**: **merge-as-is**

## Context

`LangfuseLLMObsOTELAttributes.set_messages` at `langfuse_otel_attributes.py:88-105` (pre-fix) built the OpenInference `langfuse.observation.input` attribute by `prompt = {"messages": kwargs.get("messages"), ...}` and then `json.dumps(prompt)`. The `messages` value was passed straight through. When the caller passes `list[litellm.Message]` (a pydantic model exported from the `litellm` top-level package as a documented public surface), `json.dumps` raises `TypeError: Object of type Message is not JSON serializable`, the `safe_set_attribute` chain bails on the first attribute, and *every LLM call* with a Message-typed `messages` list spams the log with:

```
LiteLLM:ERROR: _utils.py:415 - [Arize/Phoenix] Failed to set OpenInference span attributes: Object of type Message is not JSON serializable
```

Beyond noise, the PR body correctly notes the *real* impact: the trace still exports but with the input payload missing, and the span's `OPENINFERENCE_SPAN_KIND` is degraded from `GENERATION` to a generic span (because the kind attribute is set later in the same chain that bailed early). So a Langfuse trace user loses both the prompt content *and* the span-kind classification on every generation, silently, just by passing the documented Message type.

## What's right

- **The fix is at the right layer — at the boundary where pydantic objects meet `json.dumps`.** `langfuse_otel_attributes.py:89-100`: `raw_messages = kwargs.get("messages") or []` then `messages = [m.model_dump() if hasattr(m, "model_dump") else m for m in raw_messages]`. This converts `litellm.Message` (and any other pydantic v2 model with `model_dump`) to a plain dict before the JSON serializer sees it. The `hasattr(m, "model_dump")` duck-typing is the right discipline for an OTEL attribute setter that doesn't want to import `litellm.types.utils.Message` directly (which would create a circular-import risk in an integrations module).
- **The `or []` guard at `:96`** correctly handles `kwargs.get("messages")` returning `None` (caller didn't pass `messages` at all — the iteration just yields zero messages rather than `TypeError: 'NoneType' is not iterable`).
- **The comment at `:89-95`** is the right kind of in-code documentation: it names the failed primitive (`json.dumps`), names the public-API caller pattern (`list[litellm.Message]`), names the user-visible symptom (the spammy error string verbatim), *and* names the second-order impact (input payload dropped, span-kind degraded). Future readers don't have to dig the PR body out of git blame to understand why this two-line transform exists.
- **Test coverage matches the contract.** `test_langfuse_otel.py:118-152` (`test_set_messages_handles_pydantic_message_objects`) constructs `Message(role="user", content="hello")` + `Message(role="assistant", content="hi back")`, calls `LangfuseLLMObsOTELAttributes.set_messages(MagicMock(), kwargs)`, and asserts the captured `langfuse.observation.input` JSON parses cleanly into a payload with the right `role`/`content` per message. The "Must not raise — previously raised TypeError inside json.dumps" comment at `:148-149` is the regression-pin discipline.
- **The companion test at `:154-176` (`test_set_messages_passes_through_plain_dicts`)** pins the existing happy-path so the fix doesn't regress the dict-input case via an over-aggressive `model_dump` call. The plain-dict path goes through the `else m` branch of the comprehension and stays a dict.
- **Both tests use `safe_set_attribute` patching** to capture the actual attribute payloads (not just "did we crash"), which is the right level of test depth — asserting on observable output, not on internal state.

## Risks / nits

- **`hasattr(m, "model_dump")` is pydantic-v2-specific.** Pydantic v1 used `m.dict()`. If litellm still supports any caller path that produces v1 BaseModel instances, those would slip past the `hasattr` check and fall through to `json.dumps` and raise the same TypeError. A `hasattr(m, "model_dump") and callable(m.model_dump)` plus an `elif hasattr(m, "dict")` arm would close that — but litellm's own `Message` is v2 (`from litellm.types.utils import Message` in the test imports a v2 model), so this is probably fine in practice.
- **The fix only covers `messages`** in `set_messages`. The pre-fix `set_messages` body also reads `optional_params.get("functions")` and `optional_params.get("tools")` from the same `kwargs`, both of which could in principle contain pydantic-typed nested values that would fail the same way at `json.dumps` time. Probably fine since `functions`/`tools` are conventionally `list[dict]` in OpenAI-shape, but worth a follow-up grep to confirm the same Message-equivalent type doesn't exist for tool-call shapes.
- **Could share a `_to_jsonable(value)` helper** with the rest of `langfuse_otel_attributes.py` rather than inlining the comprehension, so any *other* attribute setter on the same class that writes pydantic-typed values gets the same protection automatically. Refactor not blocking.

## Verdict

**merge-as-is.** Two-line fix at the exact right layer with a honest comment that documents the failure mode. Two tests pin both the regression and the existing happy path. The fix closes a real, currently-active log spam + telemetry-degradation bug for any caller using the documented `litellm.Message` type. The nits are about scope expansion (other attribute setters, pydantic-v1 fallback, helper extraction) that should not block the fix.
