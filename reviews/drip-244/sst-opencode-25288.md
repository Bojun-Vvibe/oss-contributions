# PR #25288 — fix(mcp): truncate tool keys longer than 64 characters

- Repo: sst/opencode
- Head: `5c07032a396926681a4989a604e61ce71641b739`
- URL: https://github.com/sst/opencode/pull/25288
- Verdict: **merge-as-is**

## What lands

Closes #3523 — Anthropic and OpenAI tool-calling APIs reject names >64 chars,
which silently kills sessions when an MCP server name + tool name exceeds the
limit. Fix introduces `MAX_TOOL_NAME_LENGTH = 64` and a `buildToolKey`
helper at `packages/opencode/src/mcp/index.ts:118-128`:

```
function buildToolKey(clientName, toolName) {
  const key = sanitize(clientName) + "_" + sanitize(toolName)
  if (key.length <= MAX_TOOL_NAME_LENGTH) return key
  const suffix = "_" + Hash.fast(key).slice(0, 8)
  return key.slice(0, MAX_TOOL_NAME_LENGTH - suffix.length) + suffix
}
```

The natural `<client>_<tool>` form is used unchanged when it fits — zero
behavior change for the common case. Only when truncation is needed does the
key get the deterministic 8-char hash suffix derived from the *full* untruncated
key (so two long tools sharing a long-name prefix still get distinct keys).
The truncation site at `mcp/index.ts:670-679` also emits a `log.warn` carrying
both `clientName` and `toolName` so users can rename if they want a stable
human-readable key.

## Test coverage

`packages/opencode/test/mcp/lifecycle.test.ts:706-743` adds a regression test
using a 62-char server name plus tools `alpha`/`bravo` — asserts all keys are
≤64, all match the sanitize charset `[a-zA-Z0-9_-]+`, and the two keys are
distinct (`new Set(keys).size === 2`). PR body confirms 20/20 pass with 54
expect() calls.

## Why merge-as-is

- The contract under fix is hard-required by the upstream API (not a
  preference) and the threshold is correctly named at module scope.
- Suffix construction `_<8 hex>` is `1 + 8 = 9` chars, leaving 55 chars of
  prefix — plenty of human-readable signal for debugging.
- `Hash.fast(key)` over the full pre-truncation key (not the truncated form)
  is the correct collision-resistance choice; truncating-then-hashing would
  collide identically-prefixed names.
- Fail-open behavior preserved (no throw, just rename + warn) so a misconfigured
  MCP server doesn't take down the session — exactly the right tradeoff for the
  reported failure mode.

The author's testing notes are credible (`bun test`, oxlint, typecheck all
green).