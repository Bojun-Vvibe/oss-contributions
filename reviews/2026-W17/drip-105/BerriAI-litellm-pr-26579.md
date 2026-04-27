---
pr: https://github.com/BerriAI/litellm/pull/26579
sha: 2dbac3d8
diff: +56/-0
state: OPEN
---

## Summary

Adds `parallel_tool_calls` and `text` to the chatgpt-passthrough provider's allowlist of preserved Responses-API request params, so structured-output schemas (`text.format.json_schema`) and explicit parallel-tool-call disables survive the proxy translation instead of being silently dropped. Pure additive change to `transformation.py` plus three new parametrized tests pinning the JSON-schema, JSON-object, plain-text, and `parallel_tool_calls=False` cases.

## Specific observations

- `litellm/llms/chatgpt/responses/transformation.py:96-105` — adds two strings to the allowlist set. Order isn't semantic but the placement (`parallel_tool_calls` after `include`, `text` after `reasoning`) keeps related fields adjacent. This is the entire production change — minimal blast radius.
- `tests/test_litellm/llms/chatgpt/responses/test_chatgpt_responses_transformation.py:107-130` — `test_chatgpt_preserves_text_param` is parametrized over three realistic `text` values: a full JSON-schema with `strict: True`, a minimal `json_object`, and the plain `text` format. Each case asserts `request["text"] == text_value` (i.e. the value is preserved verbatim, not normalized) plus that the existing forced-streaming and `reasoning.encrypted_content` injection still happen. Good coverage of the "preserve, don't normalize" contract.
- `:131-147` — `test_chatgpt_preserves_parallel_tool_calls` pins the boolean preservation with `parallel_tool_calls=False`. Worth also testing `=True` to lock in that the field is faithfully passed through both directions, not only the "disable" case (the field's purpose is two-valued; testing one polarity leaves room for an accidental coercion bug on the other).
- `:182-204` — extends the existing `test_chatgpt_drops_unsupported_responses_params` parametrized test to include `parallel_tool_calls: False` and `text: {format: {type: json_object}}` in the input dict, plus matching assertions in the output. This is the test that previously *protected the bug* (asserted that *only* the documented set of params gets through); now it correctly includes the two new fields. The fact that a single existing test got the assertion deltas (vs. a separate test) means the "preserve allowlist" contract is centrally pinned, which is the right place.
- The PR adds zero changes outside this provider. The Responses-API allowlist pattern likely exists in sibling providers (`responses/transformation.py` for openai, azure, etc.) — if so, those providers might have the *same* missing-fields bug for `parallel_tool_calls` / `text`. The PR description doesn't say whether this was checked. A grep for `"parallel_tool_calls"` and `"text"` across all `responses/transformation.py` files would tell you whether this is one of N similar bugs or just chatgpt.
- No release note / changelog entry. For litellm this might land via release tooling automatically, but a "users sending `text.format.json_schema` to chatgpt-passthrough were silently losing structured-output guarantees" line is the kind of regression-severity context that downstream users absolutely need to see if they were debugging "why is GPT-5-Codex returning unstructured output even when I send a schema."
- Aside on the `text` field's structure: the test for `format.type=json_schema` includes `"schema": {...}` with `additionalProperties: False`. That means the test doubles as an integration smoke test for the Responses-API JSON-schema *contract*, not just the proxy passthrough. Worth keeping that fidelity if the test ever gets simplified — a stripped-down `{"format": {"type": "json_schema", "name": "x"}}` would technically pass the proxy assertion but stop validating the realistic shape.

## Verdict

`merge-as-is` — minimum-viable, well-scoped fix for a real silent-data-loss bug. Two optional follow-ups (not blocking): (1) add a `parallel_tool_calls=True` case to lock in symmetric preservation, (2) check whether sibling Responses-API transformations (openai/azure/anthropic-passthrough/etc.) need the same fix.

## What I learned

Allowlist-based parameter passthrough is the safest default (deny by default, opt-in supported fields), but it has a built-in failure mode: when the upstream API adds a new supported param, the proxy silently drops it until someone hits the bug, files an issue, and lands an additive PR exactly like this one. The mitigation is a periodic "diff our allowlist against the upstream OpenAPI spec" maintenance task — without it, the long tail of these one-line fixes is forever.
