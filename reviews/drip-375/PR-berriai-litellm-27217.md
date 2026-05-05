# BerriAI/litellm #27217 — fix(vertex): forward system prompt to partner model token counter

- **PR:** BerriAI/litellm#27217
- **Head SHA:** `2df51f7aa33a0d6d5e3f0967a22c4160234c922b`
- **Files:** 2 changed (+4 / -0) — minimal, surgical fix

## What changed

- `litellm/llms/vertex_ai/common_utils.py:1085` — the existing `count_tokens(...)` dispatch into the partner-model handler now also passes the previously-collected `system=system` kwarg through, alongside the existing `vertex_project / vertex_location / vertex_credentials` triplet. Without this, the system prompt was being assembled at the dispatcher level but silently dropped on the way into the partner-model path.
- `litellm/llms/vertex_ai/vertex_ai_partner_models/main.py:280` — the `count_tokens` signature gains `system=None` as the matching keyword arg. At `main.py:322-323` a guard `if system is not None: request_data["system"] = system` then injects it into the Anthropic-shaped request body. The `is not None` check (rather than truthy) means an explicitly-empty system string would still be forwarded — important for parity with how the actual completion path treats system messages.

## Risks / notes

- Anthropic's `messages` count-tokens endpoint accepts `system` as either a string or a list of content blocks. The PR forwards whatever the dispatcher built without re-shaping it. If the upstream caller is constructing a list-of-blocks form, this should "just work" because the partner endpoint already accepts that shape; if it's a stringified concatenation, ditto. No type-narrowing needed at this layer, but a typed annotation (`system: Optional[Union[str, List[dict]]] = None`) on the partner-side signature would be a tiny readability win.
- Mistral and other partner models on Vertex may have different token-count semantics for system prompts. The partner-model dispatcher in `main.py` doesn't currently fork on model family for `count_tokens`, so the same `system` kwarg gets forwarded to whatever provider the request resolves to. For Anthropic-on-Vertex this matches their contract; for Mistral-on-Vertex it may be silently ignored or cause schema validation failure depending on the Vertex Mistral spec. Worth a quick spot-check or at least a comment noting the assumption.
- No tests added. A 4-line shim like this is hard to regress, but a single unit test asserting the `system` kwarg lands in `request_data["system"]` would prevent silent breakage if someone refactors the partner-model dispatch.

## Verdict

**merge-after-nits** — correct and minimal fix for an obvious data-loss bug. Add a 1-test guard and ideally a comment about partner-model-family applicability before merging.
