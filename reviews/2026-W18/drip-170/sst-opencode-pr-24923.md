# Review: sst/opencode#24923 — feat(provider): add apiKeyCommand for dynamic API key refresh

- **PR:** https://github.com/sst/opencode/pull/24923
- **Head SHA:** `117446216a8183960d80c315f91b745d8c8eef7e`
- **Author:** darthbeeius
- **Diff size:** +31 / -0 (single file: `packages/opencode/src/provider/provider.ts`)
- **Verdict:** `merge-after-nits`

## What it does

Adds a per-provider `apiKeyCommand` option that runs a shell command to mint a fresh API key on every request, with a 5-minute in-memory TTL cache. Patterned after Claude Code's `apiKeyHelper`. Wires into the existing `customFetch` wrapper at `provider.ts:1449-1490` by:

1. Reading and deleting `apiKeyCommand` from `options` alongside `chunkTimeout` (so it doesn't leak to the SDK constructor).
2. Defining a closure `resolveApiKey()` that returns the cached value if not expired, else `Bun.spawnSync(["sh", "-c", apiKeyCommand], { timeout: 10_000 })` and caches `stdout.trim()` for 5 min. Falls back to last-known cached value on throw.
3. In the fetch override, when `apiKeyCommand` is set, overwrites both `Authorization: Bearer <k>` and `x-api-key: <k>` headers on every outbound request.

## Specific citations

- `provider.ts:1452` — `const apiKeyCommand = options["apiKeyCommand"]` + `delete options["apiKeyCommand"]` mirrors the existing `chunkTimeout` shape correctly; without the delete this would explode AI SDK constructor schemas.
- `provider.ts:1463-1475` — `Bun.spawnSync(["sh", "-c", apiKeyCommand], { timeout: 10_000 })` shells out via `sh -c`, so the value of `apiKeyCommand` is interpreted as shell. That's the same model as Claude Code's `apiKeyHelper` and intentional, but it deserves a security-note in docs.
- `provider.ts:1480-1488` — sets BOTH `authorization` and `x-api-key` unconditionally. For OpenAI-shaped providers `x-api-key` is meaningless noise; for Anthropic-shaped providers `Authorization: Bearer` is meaningless noise. Harmless on most servers but some strict gateways 400 on unknown auth headers.
- `provider.ts:1473` — empty `try/catch {}` swallows everything (`spawnSync` failure, non-zero exit code, timeout). The function then returns `cachedKey?.value` which may be a token that's already 401'd. Caller has no way to know the refresh failed.

## Nits to address before merge

1. **Add a debug log when the spawn fails.** `try {} catch (e) { log.warn(\`apiKeyCommand failed: ${e}\`) }` at `:1473`. Otherwise users see opaque 401 storms with no signal that their refresh hook is broken.
2. **Surface non-zero exit codes as failure**, not as success-with-empty-stdout. Right now `result.stdout.toString().trim()` could be empty when the command exited 1, and the code falls through to `return cachedKey?.value` silently. Check `result.exitCode === 0` explicitly.
3. **Make the 5-minute TTL configurable** (or at minimum a named constant `API_KEY_CACHE_TTL_MS` near the top of the file). Some IAM/STS tokens are 15 min, some are 1 hour; hard-coding 5 min wastes refresh calls for the latter and risks staleness near expiry for the former. A `apiKeyCommandTtlMs` option (default 300_000) is the obvious shape.
4. **Don't blindly set both header forms.** Either let the existing SDK header logic stand and only inject one based on provider id, or document that `apiKeyCommand` providers should not set static `apiKey` in config (so the SDK doesn't fight the override). At minimum, `headers.delete("x-api-key")` before `headers.set` to avoid duplicates if a previous middleware set it.
5. **No tests in the diff.** The closure-cached `cachedKey` is shared across all fetch calls for the layer, which is the right scope, but a regression test asserting (a) two requests within TTL share the same spawn, (b) a request after TTL respawns, (c) spawn failure with a still-valid cached value reuses the cache, is worth ~30 lines.
6. **Doc gap.** `apiKeyCommand` isn't in `packages/opencode/src/config/config.ts` schema or in the docs site. New users won't discover it.

## Rationale

Net +31, single localized change, follows an established Claude Code pattern, and unblocks a real enterprise pain point (IAM/STS rotation). The shape is right; the gaps are observability (silent failure), configurability (hard-coded TTL), and provider-shape correctness (dual-header injection). All fixable in a follow-up commit on the same PR.