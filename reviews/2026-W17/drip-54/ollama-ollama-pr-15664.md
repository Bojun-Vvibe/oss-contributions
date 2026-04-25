# ollama/ollama PR #15664 — openai: honor reasoning_effort in /v1/responses endpoint

@42f62397 · base `main` · +87/-0 · author `balgaly`

## Summary
Plumbs `reasoning.effort` from the OpenAI-compatible `/v1/responses` endpoint into the internal `ChatRequest.Think` field, matching existing `/v1/chat/completions` behavior. Fixes #15635.

## What changed
- `openai/responses.go:550-560` — in `FromResponsesRequest`, validates `r.Reasoning.Effort` against `{"high","medium","low","none"}`; maps `"none"` → `Think{Value:false}` and the others → `Think{Value:effort}`; surfaces invalid values as a JSON-able error.
- `openai/responses.go:567` — wires `Think: think` into the returned `*api.ChatRequest`.
- `openai/responses_test.go:1495-1567` — table-driven test `TestFromResponsesRequest_ReasoningEffort` covers all four valid values, the missing-reasoning case, and the invalid-effort error path.

## Key observations
- Validation list is hardcoded as a literal slice at line 549; minor — extracting to a package-level `validReasoningEfforts = []string{...}` would let `/v1/chat/completions` reuse it. Not blocking.
- The `errors.New`-style format string `"invalid reasoning value: '%s' (must be...)"` includes the user-supplied value verbatim — fine here since it's an OpenAI-compatible parameter, not a credential.
- Test coverage is exemplary: all branches (`none`/`low`/`medium`/`high`/missing/invalid) hit. Asserts on `chatReq.Think.Value` directly which catches the `bool vs string` polymorphism.
- Behavior parity with `/v1/chat/completions` is the right design — keeps the two OpenAI-shaped surfaces consistent for downstream clients.

## Risks/nits
- `slices.Contains` import added (line 7) — fine on Go 1.21+.
- Consider adding an end-to-end test that hits the actual HTTP handler to ensure the `Think` value survives all the way to the runner; current tests stop at `FromResponsesRequest`.

**Verdict: merge-as-is**
