# BerriAI/litellm#26426 — stop stripping output_config on Vertex Anthropic transform

**What changed.** Removes two lines (and the comment above them) in `litellm/llms/vertex_ai/vertex_ai_partner_models/anthropic/transformation.py:transform_request` that unconditionally did `data.pop("output_config", None)` before forwarding the request body to Vertex's `streamRawPredict` endpoint. The PR claims Vertex's Anthropic raw-predict endpoint now accepts and forwards `output_config` (which carries `effort` + `task_budget` controls under the `anthropic-beta: task-budgets-2026-03-13` header — a header LiteLLM was already correctly forwarding). Fixes #26423.

**Why it matters.** With the pop in place, callers setting `output_config={"effort":"high","task_budget":{...}}` got a silent feature-disabled response: no error, no warning, no truncation, the field just vanished. That's the worst class of bug — the API looks like it's working, costs accrue, and behavior degrades invisibly.

**Concerns.**
1. **No test added.** The diff is removal-only with no new test in `tests/test_litellm/llms/vertex_ai/...` asserting `output_config` survives `transform_request`. The PR description shows a manual repro but a unit test against `transform_request` would be ~10 lines and would prevent regression if a future refactor reintroduces a generic "drop unknown fields" pass.
2. **Sister field still stripped.** Line 108 still does `data.pop("output_format", None)` with the same now-suspect comment ("VertexAI doesn't support output_format parameter"). If the comment was wrong about `output_config`, it may also be wrong about `output_format`. Worth verifying against current Vertex docs in the same PR rather than fixing them one bug report at a time.
3. **Failure mode for genuinely-unsupported endpoints.** PR argues a future Vertex endpoint that rejects `output_config` will surface as a 400 — true, but that 400 will reach the user mid-stream on a streaming completion. A defensive allowlist (e.g. only forward `output_config` when the beta header is present) would scope the blast radius.
4. **No back-compat concern** — removing a silent field-strip can't break callers who weren't relying on the strip.
5. **Co-author trailer.** Body contains a generated-with trailer; project convention varies — confirm BerriAI accepts these in commit messages.

Net: correct fix; should land with a test and a same-PR audit of `output_format`.
