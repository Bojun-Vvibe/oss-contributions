# sst/opencode PR #25889 — feat(websearch): add Perplexity backend (default), keep Exa as alt

- URL: https://github.com/sst/opencode/pull/25889
- Head SHA: `916eb3aabe3d8969a202a0490442ef7d8d52015a`
- Author: jliounis (James Liounis)
- Files: `packages/core/src/flag/flag.ts`, `packages/opencode/src/tool/{mcp-perplexity.ts,websearch.ts,registry.ts}`, docs

## Assessment

The new backend is a clean, contained ~75 LOC mirror of `mcp-exa.ts` at `packages/opencode/src/tool/mcp-perplexity.ts:1-76`. It uses Effect's HttpClient + Schema for typed request/response (`PerplexityRequest`, `PerplexityResult`), reads `PERPLEXITY_API_KEY` (with `PPLX_API_KEY` fallback), sends a `User-Agent`-equivalent `X-Pplx-Integration: opencode/${InstallationVersion}` header for upstream attribution, and applies a 25-second `Effect.timeoutOrElse` matching the Exa path. The schema is intentionally narrow (`title`, `url`, optional `snippet`, optional `date`) which is exactly what's needed to render the existing `1. title \n  url \n  snippet` LLM-friendly format.

Backend selection at `websearch.ts:25-33` is gated by `usePerplexityBackend()` which double-checks both the flag *and* the env var presence — important because the flag derivation at `flag.ts:73-78` makes Perplexity the default whenever `PERPLEXITY_API_KEY` is set unless `OPENCODE_DISABLE_PERPLEXITY` is truthy. The runtime guard prevents a user who set the flag but no key from getting an `Effect.die("PERPLEXITY_API_KEY is not set")` crash. Registry change at `registry.ts:284-291` correctly ORs the new flag in so the tool surfaces when either backend is configured.

One nit on the flag definition at `flag.ts:73-78`: the `!falsy("OPENCODE_DISABLE_PERPLEXITY") && !truthy("OPENCODE_DISABLE_PERPLEXITY")` pair is redundant — `falsy` and `truthy` are typically opposites for the same env var, so `!truthy("OPENCODE_DISABLE_PERPLEXITY")` alone is what's intended. Keeping both means an unset `OPENCODE_DISABLE_PERPLEXITY` (where `falsy` returns false and `truthy` returns false) still passes, which is the right default behavior, but the `!falsy(...)` clause is dead. Worth simplifying or commenting why both are needed. Also: `contextMaxCharacters` at `mcp-perplexity.ts:64-66` truncates by character count but the result body is multi-paragraph so a mid-line slice can leave a partial URL — minor cosmetic issue but worth a `lastIndexOf("\n\n")` boundary cut.

The Exa-only annotations on `livecrawl` and `type` parameters at `websearch.ts:13-21` are a nice UX touch so the LLM doesn't waste tokens setting parameters that get silently ignored. Documentation updates in `tools.mdx` and `providers.mdx` are out-of-scope for the diff snippet shown, but the PR summary indicates the precedence is documented.

## Verdict

`merge-after-nits`
