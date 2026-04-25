# ollama/ollama PR #15683 — preserve thinking in /api/generate; populate parameter_size in /api/tags

@5ef5c1cc · base `main` · +40/-16 · author `serenposh`

## Summary
Two independent bug fixes against `server/routes.go`: (1) reorder parser init in `GenerateHandler` so `req.Think` defaults are applied before `parser.Init`; (2) backfill `Family` / `ParameterSize` / `QuantizationLevel` for safetensors models in `/api/tags` so it matches `/api/show`.

## What changed
- `server/routes.go:374-407` — moves the `builtinParser.Init(nil, nil, req.Think)` block to after the capability-resolution step so `req.Think` is the resolved value (mirrors `ChatHandler` ordering). New comment at line 396-398 explains why.
- `server/routes.go:1430-1473` — extracts `details := api.ModelDetails{...}` then, for `cf.ModelFormat == "safetensors"` with `completion` capability, calls `xserver.GetSafetensorsLLMInfo(n)` to fill `Family` from `general.architecture` and `ParameterSize` from `general.parameter_count`; falls back to `GetSafetensorsDtype` for quantization.

## Key observations
- The Gemma4 fix is well-scoped: only ordering of the existing `builtinParser` block changes, and it now matches the documented behavior of `ChatHandler`. Low risk.
- Safetensors backfill duplicates logic that lives in `ShowHandler`; consider extracting a `safetensorsDetails(n, cf)` helper used by both handlers to keep `/api/show` and `/api/tags` from drifting again.
- `format.HumanNumber(uint64(paramCount))` — `paramCount` is `int64`; if a malformed safetensors header reported a negative count this would silently produce a huge number. Worth a `paramCount > 0` guard (the diff already has it at line 64 — good).
- Two-bug-in-one-PR is fine here since both touch the same file and same family of model, but the commit message should make it splittable later.

## Risks/nits
- Should add a regression test for `/api/generate` thinking with `gemma4`-class models; the fix is otherwise invisible to CI.

**Verdict: merge-after-nits**
