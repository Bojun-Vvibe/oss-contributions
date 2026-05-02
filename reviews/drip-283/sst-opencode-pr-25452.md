# sst/opencode PR #25452 — fix(telemetry): emit Tool.execute span for MCP and plugin tools

- Head SHA: `a626be423816402e09e1c7f8717cb70414a7bdab`
- Size: +22 / -4, 2 files

## Specific refs

- `packages/opencode/src/session/prompt.ts:467-481` — wraps the MCP execute path (`ctx.ask` + `Effect.promise(() => execute(args, opts))`) in `Effect.gen` then pipes through `Effect.withSpan("Tool.execute", ...)` with `tool.name`, `tool.call_id`, `session.id`, `message.id`. Plugin `tool.execute.before/after` hooks deliberately stay outside the span — matches the native scope where `wrap()` covers decode + execute + truncate only.
- `packages/opencode/src/tool/registry.ts:157-166` — pipes `fromPlugin()`'s `Effect.gen` body through the same `withSpan`, so user-defined tool files and plugin-supplied tools also get coverage. `tool.call_id` is conditionally included (`...(toolCtx.callID ? ... : {})`) since plugin context may not always have one.
- Evidence in PR body: trace `33c8941c89094d50650c2b91e244c778` had 28 `ai.toolCall` spans but only 24 `Tool.execute` children — the 4 missing were exactly the MCP tools (`sentry_search_issues`, `sentry_find_organizations`, `sentry_whoami`, `exa_web_search_exa`). `Permission.ask` fired 28 times so gating was fine; only the execution wrapper was bypassed.

## Assessment

Real observability gap with concrete trace evidence. The fix mirrors the attribute shape from native `wrap()` exactly, which is what you want for downstream filtering on `operation=Tool.execute`. Including `ctx.ask` inside the span (line 468 vs 467) is a deliberate scope choice: it captures permission-prompt latency as part of the tool execution, which slightly broadens the native semantic (where `wrap()` doesn't include `ask`) — worth a brief note in the commit message but not wrong. Typecheck and 7/7 registry/tool-define tests pass.

verdict: merge-after-nits
