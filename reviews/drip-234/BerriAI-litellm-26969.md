# BerriAI/litellm#26969 — chore(guardrails): tighten tool permission checks

- **PR**: https://github.com/BerriAI/litellm/pull/26969
- **Head SHA**: `5e7c9e68e67fa58dc88b26cc3a016cbfa4eb6c4a`
- **Size**: +371 / -36, 2 files
- **Verdict**: **merge-after-nits**

## Context

Tool-permission guardrail had three real gaps before this PR:
1. **Legacy OpenAI `function_call` shape ignored.** Both pre-call (request `functions: [...]` + `function_call: {...}`) and post-call (response `choice.message.function_call`) were not seen by the guardrail. A user calling the legacy Chat-Completions function-call surface could bypass the entire permission layer.
2. **Forced-tool-choice ignored.** `tool_choice: {"type":"function","function":{"name":"X"}}` was not enumerated as a tool the request "uses", so a `tool_choice` forcing a denied tool wasn't caught at request time.
3. **Argument parse failures fell open.** `_parse_tool_call_arguments` returned `{}` on `JSONDecodeError` — which then matched `if not arguments` → the rule entered `last_pattern_failure_msg = ...; continue`, meaning a parse failure on a rule with `allowed_param_patterns` could fall through to a *later* rule that didn't have param patterns and *allow* the call. Fail-open on malformed input.

## Design analysis

### `_parse_tool_call_arguments` (`tool_permission.py:225-250`)

Return-type widened from `Dict[str, Any]` to `tuple[Optional[Dict[str, Any]], Optional[str]]`. Three failure cases now return `(None, reason)`:
- Empty/missing arguments → `(None, "missing arguments")`
- `JSONDecodeError` / `TypeError` → `(None, "arguments could not be parsed")`
- Non-string non-dict, or parsed-but-non-dict → `(None, "arguments must be a JSON object")`

Critically, the success case is `(parsed_arguments, None)`. Old `verbose_proxy_logger.debug` "Ignoring non-dict arguments" → "Rejecting non-dict arguments" — the verb now matches the behavior.

### `_get_permission_for_tool_call` (`:330-372`)

Old shape on parse failure:
```python
arguments = self._parse_tool_call_arguments(tool_call)  # {} on failure
if not arguments:
    last_pattern_failure_msg = ...  # SOFT — falls through to next rule
    continue
```

New shape:
```python
arguments, parse_error = self._parse_tool_call_arguments(tool_call)
if parse_error:
    return False, rule.id, message  # HARD DENY — same rule
if not arguments:
    return False, rule.id, message  # HARD DENY — same rule
```

This is the load-bearing fix-open → fail-closed change. Parse failure or empty-args on a rule that requires param patterns now denies *immediately* with the failing rule's id, rather than falling through to a more permissive sibling rule. Both branches also route the message through `self.render_violation_message(default=..., context={...})` so user-templated violation messages get the same treatment as the success path.

### Legacy `function_call` shape (`:387-401`)

New `_legacy_function_call_to_tool_call(function_call, choice_index)` returns a synthetic `ChatCompletionMessageToolCall` with `id = f"legacy_function_call_{choice_index}"`, `type="function"`, `function={"name", "arguments"}`. The synthetic id pattern is the right choice — stable per-choice, doesn't collide with the LLM-supplied tool call ids (those are typically `call_<hex>`), and round-trips for the response-modification path that needs to look up the error result.

`_extract_tool_calls_from_response` at `:411-422` now iterates `enumerate(response.choices)` and appends both the new-style `tool_calls` and the synthetic legacy one. Symmetrically, `_modify_response_with_permission_errors` at `:586-611` looks up `error_results.get(legacy_tool_call.id)` and, if present, sets `choice.message.function_call = None` and appends the error message to content.

Test arm `test_extract_tool_calls_legacy_function_call_format` at `test_tool_permission.py:223-242` pins the contract for the response side.

### Pre-call request enumeration (`:425-510`)

New helpers replace the old `data.get("tools")`-only walk:
- `_get_request_tool_name(tool)` — handles dict-or-attr `function.name` extraction.
- `_get_legacy_function_name(function)` — for `data.get("functions")`.
- `_get_named_tool_choice(data)` — handles string `"foo"`, `{"type":"function","function":{"name":"foo"}}`, returns None for `"auto"|"none"|"required"`.
- `_get_named_function_call(data)` — same for legacy `function_call`.
- `_collect_request_tools(data)` — unions all four sources into `List[tuple[str, Optional[str]]]`.

`async_pre_call_hook` at `:644-662` now drives the loop with `_collect_request_tools(data)` and the warning message is updated to "No tools or functions in data".

`_modify_request_with_permission_errors` at `:483-525` now strips denied tools from all four surfaces:
- `data["tools"]` — function-typed tools matching denied set
- `data["functions"]` — legacy
- `data["tool_choice"]` — set to `"none"` if it forces a denied tool
- `data["function_call"]` — set to `"none"` if it forces a denied tool

The `tool_choice → "none"` choice is the right call vs. `del data["tool_choice"]` — "none" is the explicit OpenAI signal that the model should not call any tool, which matches the user intent of "you are not allowed to call this".

### Helper `_get_mapping_value` (`:374-378`)

Centralizes the "dict.get(key) or attr(key) or None" pattern. Used 8+ times across the new helpers — without it, each helper would repeat the dict-vs-object branching. This is a small but important DRY move because litellm's request shapes come from both pydantic models and raw dicts depending on the entry point.

## What's right

- **Fail-closed on parse error.** The single most important behavior change in the PR — turns a silent fall-through-to-next-rule into a hard deny on the failing rule. Includes a parse-error-specific message so debugging is easy.
- **Four-surface request enumeration.** `tools` + `functions` + `tool_choice` + `function_call` — exhaustive coverage of the OpenAI request shapes that can name a tool. Not just `tools`.
- **Legacy `function_call` bidirectionally handled.** Both request-time enumeration and response-time error stripping. The synthetic id pattern (`legacy_function_call_{choice_index}`) keeps the existing `error_results: Dict[id → result]` plumbing intact.
- **Centralized `_get_mapping_value`** for the dict-or-attribute extraction pattern.
- **Render-violation-message path used in both new fail-closed branches** so user-templated messages stay consistent.

## Nits / discussion

- **Synthetic id collision risk.** `legacy_function_call_{choice_index}` could collide with a real LLM tool_call_id of the same string if a model ever produces `id="legacy_function_call_0"`. Astronomically unlikely but a UUID-prefix or `__litellm_legacy_function_call_{choice_index}` would be bulletproof.
- **`tool_choice="required"` short-circuit returns None** in `_get_named_tool_choice` at `:467-468`. Correct (no specific tool named) but worth a one-line PR-body or code comment confirming the intent — `"required"` means "must call *some* tool" which is a different shape from naming a specific one.
- **`functions` deprecation context.** OpenAI deprecated the legacy `functions`/`function_call` surface in 2024; supporting it now is correct because consumers still send it, but a `# Legacy OpenAI Chat Completions surface, deprecated 2024` comment block at the top of the new helpers would orient future readers.
- **No test arm for the fail-closed parse-error path.** The diff adds the legacy-`function_call` test but I don't see a test pinning that `_get_permission_for_tool_call` now hard-denies on `JSONDecodeError` rather than falling through. Given this is the highest-impact behavior change in the PR, it should have its own `test_parse_error_denies_immediately` arm asserting `(False, rule_id, message_containing_parse_error)` rather than `(True, ...)` from a downstream permissive rule.
- **`new_tools` shadowing.** In `async_pre_call_hook` the variable is now `List[tuple[str, Optional[str]]]` rather than the historic `List[ChatCompletionToolParam]`. Same name, different type. A rename to `request_tool_names_and_types` (matching `_collect_request_tools` return) would make the type change obvious to a reader skimming.
- **Choice-iteration `choice_index`** is now used in both extract and modify paths — verify the legacy `function_call` id is generated *consistently* in both directions (extract uses `choice_index`, modify also uses `choice_index`, both via `enumerate(response.choices)` — confirmed in diff). Worth a one-line comment "id stability across extract/modify is intentional — keyed by choice index" so a refactor doesn't break it.

## Verdict

**merge-after-nits** — high-value security tightening that closes three real bypass paths (legacy `function_call`, forced `tool_choice`, fail-open on parse error). The fail-open → fail-closed change is the most important fix and deserves its own test arm before landing. Synthetic-id pattern is correct but could be made bulletproof with a non-collidable prefix.
