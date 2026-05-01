# BerriAI/litellm#26969 — chore(guardrails): tighten tool permission checks

- **PR**: https://github.com/BerriAI/litellm/pull/26969
- **Author**: stuxf
- **Head SHA**: `8538193bd336109c617a070e27918f63e62e480a`
- **Files (top)**: `litellm/proxy/guardrails/guardrail_hooks/tool_permission.py`, `litellm/proxy/_lazy_openapi_snapshot.py`, `litellm/proxy/_lazy_openapi_snapshot.json` (+ tests)
- **Verdict**: **merge-after-nits**

## Context

Two unrelated tightenings bundled:

1. **Tool-permission guardrail behavior fix.** Pre-fix `_parse_tool_call_arguments` returned `{}` on every failure mode (missing arguments, JSON decode error, non-dict result). Caller at `_get_permission_for_tool_call` then treated empty `{}` as `last_pattern_failure_msg = "missing arguments..."` and *continued* to the next rule — meaning a malformed tool call could silently skip the rule and potentially match a permissive fallback. Plus, legacy OpenAI `function_call` (single-call shape, deprecated but still emitted by some clients) wasn't covered at all.
2. **OpenAPI snapshot stability.** `_lazy_openapi_snapshot.py` produces a snapshot for diffing API surface changes. FastAPI derives the default `operationId` suffix from `set` iteration over `route.methods`, which is process-nondeterministic — so the snapshot drifted between regenerations even when no routes changed.

## What's right

- **Three-state return from `_parse_tool_call_arguments`** (`tool_permission.py:225-258`): now returns `tuple[Optional[Dict], Optional[str]]` with an explicit reason string. Three failure modes → three distinct messages: `"missing arguments"`, `"arguments must be a JSON object"`, `"arguments could not be parsed"`. The caller at `:333-350` distinguishes "parse_error" (immediate `False, rule.id, message` fail-loud) from "no arguments" (also fail-loud now). The prior "continue to next rule" behavior on parse failure is gone — correct fail-closed direction at the trust boundary.
- **`render_violation_message` plumbed through both arms.** `:339-348` passes `context={"tool_name": tool_identifier, "rule_id": rule.id}` so operators can override the default error format via the standard guardrail message-template surface.
- **Legacy `function_call` coverage** via new `_legacy_function_call_to_tool_call` at `:393-411` synthesizes a `ChatCompletionMessageToolCall` from the deprecated `message.function_call` shape, with a synthetic ID `legacy_function_call_{choice_index}` that's stable per-choice. Closes a real bypass surface (clients still emitting the legacy single-call shape would skip permission checks entirely under the prior code).
- **`_get_mapping_value` helper at `:381-385`** handles both dict and object access (`item.get(key)` for dicts, `getattr(item, key, None)` for typed objects) — correct because Pydantic-vs-dict drift across the litellm types module means call sites can land either shape.
- **OpenAPI snapshot determinism via `_normalize_operation_ids`** at `_lazy_openapi_snapshot.py:29-61`: walks every path's operations, finds the set of HTTP methods registered on that path, and rewrites each `operationId` so its suffix matches the actual method (`anthropic_proxy_route_anthropic__endpoint__put` → `..._delete` / `..._get` / `..._patch` / `..._post` per actual method). The snapshot diff shows this fix is load-bearing — pre-fix every multi-method route had all four methods sharing the same `_put`-suffixed operationId. The `HTTP_METHODS = {"delete", "get", "head", "options", "patch", "post", "put"}` set at `:16` is the correct seven-element FastAPI surface.

## Risks / nits

- **`_legacy_function_call_id(choice_index)` cardinality**: one synthetic ID per choice. If a client somehow emits the legacy shape across multiple choices in one response, the synthetic IDs `legacy_function_call_0`, `legacy_function_call_1` etc. don't collide — fine. But if downstream logging keys on tool-call ID for dedup, the `legacy_function_call_*` prefix would benefit from a comment pinning that it's intentionally distinguishable from real OpenAI-emitted IDs (which are typically `call_<32-hex>`).
- **`_normalize_operation_ids` mutates in place via `path_ops.values()` iteration while assigning to `operation["operationId"]`** — safe because dict mutation during `.values()` iteration only breaks if you add/remove keys, not if you mutate values, but a comment confirming "we mutate in place but don't change keyset" would help future readers.
- **No test pinning operationId stability across processes.** A `test_normalize_operation_ids_is_deterministic` that constructs a multi-method route, calls `generate_snapshot()` twice in the same process, and asserts equality would pin the fix. Stronger version: spin up two subprocesses and compare — but in-process is probably enough since the nondeterminism source is `set` iteration order.
- **Tool-permission rule-iteration semantics change is worth a CHANGELOG mention.** Previously, "missing arguments + multiple rules" would `continue` and try the next rule (potentially matching a more-permissive rule). Now it fail-loud-aborts on the first rule that requires arguments. This is the right direction but a behavior change for any operator who was relying on the prior continue-to-next semantics.

## Verdict

**merge-after-nits.** Two genuine bypass-class fixes (silent-ignore-on-parse-failure, legacy function_call uncovered) and one drift-class fix (operationId nondeterminism), all at the right surface. Wants the operationId-stability test and a CHANGELOG note for the rule-iteration semantics change.
