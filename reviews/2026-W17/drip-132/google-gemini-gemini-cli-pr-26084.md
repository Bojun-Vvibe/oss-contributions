# google-gemini/gemini-cli PR #26084 — Fix 400 error when more than 128 tools are enabled

- **PR**: https://github.com/google-gemini/gemini-cli/pull/26084
- **Author**: @gemini-cli-robot
- **Head SHA**: `35589570bef9565b918f9c2d0894ba3e33ccdf80`
- **Size**: +230 / −8 across 4 files

## Summary

Adds a hard 128-tool ceiling at *two* layers: (1) `ToolRegistry.getFunctionDeclarations()` in `tool-registry.ts` now invokes a new private `smartLimitTools()` that prioritizes built-in tools first, then non-MCP discovered tools, then fairly distributes remaining slots round-robin across MCP servers; (2) `GeminiChat` in `geminiChat.ts` defensively re-truncates `config.tools` inside `generateContent()` if any caller still constructs a `tools` array exceeding 128 function-declarations. Two test files cover both layers — `tool-registry.test.ts` exercises a 30/60/60 (built-in/server1/server2) registry expecting `30 + 49 + 49 = 128`, and `geminiChat.test.ts` exercises a 150-declaration single-tool case expecting truncation to exactly 128.

## Verdict: `merge-after-nits`

The two-layer enforcement is the right shape — registry-level smart prioritization is the proper canonical place, but the `geminiChat` last-line-of-defense at `:653-695` covers the "library consumer constructs a config.tools manually" case where the registry isn't on the path. The smart-limit prioritization is well-motivated (built-in tools are higher signal-to-noise per slot than the 95th MCP tool from a third-party server) and the round-robin distribution across MCP servers is the right fairness semantics. But three nits deserve attention before merge.

## Specific references

- `packages/core/src/tools/tool-registry.ts:670-678` — registry-side enforcement. `MAX_TOOLS = 128` is a magic number declared inline; should be hoisted to a named module-level constant so the same value lives in one place across this file and `geminiChat.ts`. Currently the two layers each declare their own `MAX_TOOLS = 128` (registry at `:670`, chat at `:653`); a future change to 256 has to remember to update both. Extract to `packages/core/src/core/constants.ts` or similar.

- `packages/core/src/tools/tool-registry.ts:720-771` — `smartLimitTools()` is the load-bearing prioritization. The implementation correctly partitions into `builtIn` / `discovered` / `mcp`, hard-includes built-in + discovered (with truncation if those alone exceed 128 — the `result.slice(0, limit)` at `:737` is the correct fail-safe), then round-robin distributes the remainder by `serverName`. The `mcpByServer` Map and the nested `while/for` round-robin at `:752-768` are correct: it walks server-by-server picking the `index`-th tool from each, incrementing `index` only after a full pass, so server1[0] → server2[0] → server3[0] → server1[1] → ... and stops cleanly when `added >= remaining` or no server has more tools to give. Verified by the test at `tool-registry.test.ts:946-947` asserting `s1Count === 49 && s2Count === 49`.

- `packages/core/src/tools/tool-registry.ts:738-740` — when `result.length >= limit` (built-in + discovered already saturate), the function returns `result.slice(0, limit)` *without* round-robin fairness. This means if a user has 130 built-in tools (currently impossible but plausible if discovered-tools count grows), MCP servers get *zero* slots even though the user explicitly enabled them. Worth a follow-up that reserves a small floor (e.g. min 16 slots) for MCP if any MCP server is registered, with the floor coming out of the discovered-tools bucket. Not blocking for the immediate fix.

- `packages/core/src/core/geminiChat.ts:653-695` — chat-level enforcement. The `MAX_TOOLS = 128` here is a separate magic number (see nit above). The truncation logic walks `config.tools` (which is an array of `Tool` objects each potentially containing many `functionDeclarations`) decrementing `remainingSlots` per declaration; the load-bearing `if (remainingSlots <= 0) break` at `:684` correctly stops mid-array but the preceding `limitedTools.push(tool as Tool)` for non-functionDeclaration tools at `:680-682` doesn't decrement `remainingSlots` because non-function-declaration tools (e.g. retrieval / code-execution shaped tools) don't count against the 128-function-declaration limit per the Gemini API contract. Correct discrimination, but worth a one-line comment explaining *why* the non-decrement.

- `packages/core/src/core/geminiChat.test.ts:1153-1198` — `should truncate tools to 128 as a safety net in generateContent`. Constructs 150 single-declaration `Tool` wrappers (so 150 `functionDeclarations` total), exercises the path, asserts `totalFunctions === 128`. Strong test for the bare-truncation behavior. **But**: the test doesn't assert *which* 128 survive — a future change that randomly samples instead of truncating-from-the-front would still pass this test. Add a follow-up assertion like `expect(callConfig?.tools?.[0]?.functionDeclarations?.[0]?.name).toBe('tool_0')` to pin first-N-wins behavior.

- `packages/core/src/tools/tool-registry.test.ts:892-948` — `should limit tools to 128 and prioritize built-in tools`. Strong shape: asserts (1) total declarations === 128, (2) all 30 built-in tools survive (asserts each `builtin_${i}` for i in 0..29), (3) MCP tools are split 49+49 between two servers. The math comment at `:937` (`128 - 30 = 98 MCP slots, 98 / 2 = 49 per server`) is the right kind of in-test arithmetic for a future reader. Strong regression anchor.

## Nits (not blocking)

1. Two `MAX_TOOLS = 128` magic numbers (registry + chat). Hoist to one shared constant.
2. `smartLimitTools` has no behavior when `tools.length <= limit`; it's correctly only called inside the `if (activeTools.length > MAX_TOOLS)` guard at `:678`, but a future caller that forgets the guard would get unexpected reordering. Add an early return `if (tools.length <= limit) return tools;` at the top of `smartLimitTools` for defensive symmetry.
3. The `debugLogger.warn` calls at `tool-registry.ts:680-682` and `geminiChat.ts:665-667` are visible only in debug mode. The user-visible signal that they're losing tools they enabled is silent — a one-line user-facing notification on the first truncation per session would be valuable, especially given the user explicitly opted into the truncated tools.
4. Truncation order for the chat-layer (`tool-registry`-bypass) safety net is "first-N functionDeclarations per first-N Tool wrappers" with no smart prioritization. If the safety net actually fires (i.e. someone bypassed the registry), they get neither the built-in nor the round-robin guarantees. Worth either threading `smartLimitTools` through to this layer or documenting the safety-net asymmetry as deliberate.
