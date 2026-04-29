# sst/opencode#24871 — fix(mcp): truncate tool names exceeding 64-char provider limit

- PR: https://github.com/sst/opencode/pull/24871
- Head SHA: `7877858011b2d063dcd4f6ddf020402e8d34e0a1`
- Author: georgeglarson
- Diff: +98/-1 across `packages/opencode/src/mcp/index.ts` and `packages/opencode/test/mcp/lifecycle.test.ts`
- Issue: #3523

## What changed

Adds a name-truncation helper at `packages/opencode/src/mcp/index.ts:118-134`. Previously the MCP layer keyed tools as `sanitize(clientName) + "_" + sanitize(toolName)` directly; OpenAI's tool-use API rejects `tools[].function.name` strings longer than 64 chars and the resulting 400 silently killed opencode (the error never surfaced in the TUI). The new `buildToolName` checks the combined length against `MAX_TOOL_NAME = 64`, and on overflow takes a content-derived 8-char `Hash.fast(combined).slice(0, 8)` and packs it as `combined.slice(0, 64 - 8 - 1) + "_" + hash`, plus a `log.warn` at truncation time. The single call site at line 666 of the same file (`result[buildToolName(clientName, mcpTool.name)] = convertMcpTool(...)`) is the only consumer.

The test surface in `lifecycle.test.ts:786-863` adds two cases: one constructs a 35-char server name `"chrome-devtools-aaaaaaaaaaaaaaaaaaa"` plus two tool names that share their first 55 chars after the prefix, and asserts `keys.length === 2`, every `key.length <= 64`, the keys remain distinct, and each ends with the `/_[0-9a-f]{8}$/` suffix. The second pins the no-op shape: short names like `"short_ping"` are not rewritten.

## Observations

The collision-avoidance argument is the load-bearing one and the test directly exercises it. Hashing the **full** combined name (not just the prefix that gets dropped) is the correct choice — two tools whose pre-hash prefix collides will still get different hashes because their *full* names differ in the suffix. If the hash were computed on `combined.slice(0, 64 - 9)` that would defeat the test. Worth a one-line code comment near the `Hash.fast(combined)` call making this explicit, since a future reader optimizing the helper might "fix" it.

The `Hash.fast` import lands at `index.ts:16`; the rest of the file uses the `@opencode-ai/core/util/*` namespace so this is consistent. `log.warn` is the right level — silent truncation would be wrong (this affects which tool the model invokes via name disambiguation), `error` would be misleading (the truncation is the recovery, not the failure).

One sharp edge: the predicate `combined.length <= MAX_TOOL_NAME` uses JS string `.length` which counts UTF-16 code units, not bytes. If a tool/server name contains supplementary-plane chars the budget calculation is wrong against an underlying byte/codepoint provider limit. In practice MCP tool names are ASCII (`sanitize(s)` strips to `[a-zA-Z0-9_-]`), so this is safe — but worth a one-line comment because `sanitize` already does the constraining work and the relationship is non-obvious.

The 8-char hex suffix gives 32 bits of entropy, ~4M distinct values; with realistic per-server tool counts (<200) collision risk is essentially zero, but if the same prefix collides across *many* servers, that's a different namespace and the `<server>_` prefix already disambiguates upstream. Acceptable.

## Verdict

`merge-after-nits`
