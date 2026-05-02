# Review: sst/opencode #25404 — fix: pass mcp_timeout as AI SDK streamText stepMs

- Repo: sst/opencode
- PR: #25404
- Head SHA: `2aecd31ed56cd49e9e08711aa51aa1dd01c21215`
- Author: claude-code-best
- Base: `dev`
- Size: +3 / -0 across 1 file

## What it does
Wires the user-configurable `experimental.mcp_timeout` into the AI SDK `streamText`
call so per-step model invocations are no longer hard-capped at the SDK's
default 120s when MCP tools take longer.

## File-level notes
**`packages/opencode/src/session/llm.ts` @ L334+ (head `2aecd31`)**
```ts
return streamText({
  timeout: cfg.experimental?.mcp_timeout
    ? { stepMs: cfg.experimental.mcp_timeout }
    : undefined,
  onError(error) { ... }
})
```
- The conditional spread (`undefined` when unset) is correct — passing
  `{ stepMs: undefined }` would override the SDK default unintentionally, this
  avoids that.
- Naming concern: the config key is `mcp_timeout` but `stepMs` applies to *any*
  step, not only MCP tool calls. The semantics drift slightly — if a user sets
  `mcp_timeout: 600_000` to tolerate a slow MCP server, they are also extending
  the per-step timeout for plain LLM streaming. Worth a one-line doc comment
  next to the field saying "also raises streamText stepMs ceiling".
- No unit added; OK for a 3-line plumb but a smoke assertion on the
  `streamText` mock options would prevent regression.

## Risks
- Low. `undefined` preserves prior behavior. Worst case is users with
  pathologically high `mcp_timeout` now inherit a long stream timeout — that
  is the intent.

## Verdict: `merge-after-nits`
Add a brief comment on the config field clarifying the broader scope, or
rename to something like `step_timeout_ms` with `mcp_timeout` as a deprecated
alias. Otherwise good.
