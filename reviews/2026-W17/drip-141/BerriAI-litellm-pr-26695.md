# BerriAI/litellm#26695 — fix(tools): resolve legacy `definitions` $ref in tool schemas for Anthropic and Fireworks

- **PR**: https://github.com/BerriAI/litellm/pull/26695
- **Author**: @hxrikp1729
- **Head SHA**: `f1ab7da89df8ea1e9cd063ea011b198ca540c1df`
- **Base**: `main`
- **State**: OPEN
- **Scope**: **+6284 / -1146 across 63 files** (the actual fix is ~50 lines + tests; rest is bad rebase)

## Summary

The substantive fix addresses a real and well-diagnosed bug: MCP servers (DevRev cited explicitly) emit JSON Schema tool parameters using draft-04-style `definitions: {...}` plus `$ref: "#/definitions/X"` pointers. Anthropic supports `$defs` (draft 2019-09+) but not `definitions`. The existing flow at `litellm/llms/anthropic/chat/transformation.py:_map_tool_helper` filters tool schemas through `AnthropicInputSchema.__annotations__` (the `_allowed_properties` set), which strips the `definitions` key while leaving the `$ref` strings intact — Anthropic's API then returns a `PointerToNowhere` error because the references point at nothing. Fireworks has the symmetric problem at its own `_transform_tools`. Fix is to call `unpack_defs` (existing utility in `prompt_templates/common_utils`) on the schema, merging both `definitions` and `$defs` into a single defs dict, before the filter strips them.

**The PR as filed is a 63-file bad-rebase that includes XecGuard guardrail (1904-line test + 585-line implementation + 314-line docs + dashboard logo), `expired_ui_session_key_cleanup_manager` brand-new module + 348-line test, predibase chat handler refactor (212+/7-), pass-through guardrail surface, ollama chat transformation, redis test churn, Bedrock anthropic_claude3 cache_control extension, and ~10 other unrelated drops.** Without a clean rebase, this is unreviewable as a tool-schema fix.

## Diff anchors (for the actual fix scope)

- `litellm/llms/anthropic/chat/transformation.py:448-471` — inside `_map_tool_helper`, before the `_allowed_properties = set(AnthropicInputSchema.__annotations__.keys())` filter at `:472`, adds:
  - `if "definitions" in _input_schema:` guard so the path is no-op for the common case.
  - `import copy` + `_input_schema = copy.deepcopy(_input_schema)` at `:457-458` — load-bearing because the schema is the caller's owned object; in-place mutation here would silently strip `definitions` from the upstream tool definition on every request.
  - Merge dict `defs = {**_input_schema.pop("definitions", {}), **_input_schema.pop("$defs", {})}` at `:464-466` — `$defs` wins on collision, which matches the JSON Schema 2019-09+ migration story (the modern key takes precedence).
  - `if defs: unpack_defs(_input_schema, defs)` at `:468-469` — guards the inline call against the empty-defs case where both keys exist with empty values.
- `litellm/llms/fireworks_ai/chat/transformation.py:200-227` — symmetric fix at `_transform_tools`. Uses the same `deepcopy → pop both keys → merge → unpack_defs` pattern. The `import copy` is hoisted to the function top at `:202` (Anthropic version inlines it inside the conditional), which is more conventional Python style. **Worth normalizing one way or the other across both call sites.**
- `tests/llm_translation/test_anthropic_completion.py:2969-3082` — `test_anthropic_tool_with_legacy_definitions_ref_resolved_inline` uses a real-shape MCP tool input (`_gen:tags` ref pointing into `definitions`), then asserts at `:3019`: `assert "definitions" not in input_schema` (the filter still strips it, because that's the existing post-filter contract) and `:3020`: `assert input_schema["properties"]["tags"]["items"]` is the inlined object (the actual `unpack_defs` outcome). Both invariants are load-bearing — the first proves we didn't widen the filter, the second proves we resolved the ref before the filter ran.
- `tests/llm_translation/test_anthropic_completion.py:3084-...` — `test_fireworks_transform_tools_resolves_definitions_refs` mirrors the Anthropic test against the Fireworks path. Symmetric coverage is the right shape.
- `litellm/llms/bedrock/chat/converse_transformation.py:1302, 1370` — drive-by `_bedrock_tools_pt(regular_tools)` → `_bedrock_tools_pt(regular_tools, model=model)` parameter threading at two call sites. Plausibly required by a sibling change downstream, but it is **not** part of the `definitions`-$ref fix story and should not ship in the same PR.
- `litellm/llms/bedrock/messages/invoke_transformations/anthropic_claude3_transformation.py:159-165` — adds `tools` to the cache_control sanitization sweep. Again, not part of the stated fix; this is a separate hygiene patch.

## What I'd push back on (request-changes)

1. **Rebase to a clean diff containing only the four files: anthropic transformation, fireworks transformation, anthropic test, fireworks test.** Everything else — XecGuard (1904+585+314 lines), expired_ui_session_key_cleanup, predibase chat handler refactor, ollama chat transformation, dashboard logo `mseep` SVG addition, model-pricing JSON 32+/32- diffs, SpendLogsSettingsModal 484+156 deletions, etc. — is unrelated work that bad-rebased into this PR. None of it is necessarily wrong, but bundling it makes the actual fix unreviewable.
2. **Verify the XecGuard inclusion is not a known-bad merge.** XecGuard is a third-party guardrail with 585 lines of new code under `litellm/proxy/guardrails/guardrail_hooks/xecguard/xecguard.py` and a 314-line docs page. If those are vendor-contributed code being smuggled into this PR's diff via a bad rebase, that is a supply-chain concern worth flagging to maintainers separately.
3. **Confirm whether this PR re-regresses the W17 drip-136 #26677 supply-chain mitigation** (`.npmrc min-release-age` change) the way #26688 (drip-140) did. A `git diff base..HEAD -- .npmrc` should be the first check before any merge.
4. **Normalize the `import copy` placement** between the Anthropic (inline-inside-conditional) and Fireworks (function-top) sites. Pick one.
5. **Add a test cell for `$defs` + `definitions` collision** — the merge dict at Anthropic `:464-466` makes `$defs` win on key collision, but no test pins that ordering. A schema with `{"definitions": {"X": A}, "$defs": {"X": B}}` and a `$ref: "#/$defs/X"` reference (or `#/definitions/X`?) would surface the collision behavior.
6. **The MCP-source attribution** (DevRev cited in the inline comment at `:451-454`) is good context, but consider linking the upstream issue/MCP spec section in the PR body so future readers can find the JSON Schema draft history without re-discovering it.

## Verdict

**request-changes** — the fix as scoped (Anthropic + Fireworks `definitions`-resolution-before-filter) is correct, well-diagnosed, and well-tested. But the PR as filed bundles 60+ unrelated files including a third-party guardrail and several unrelated transformer refactors. Recommend rebase to clean diff containing only the 4 fix files, then this is straightforward `merge-as-is` on the clean version.

Repo coverage: BerriAI/litellm (LLM provider transformer correctness).
