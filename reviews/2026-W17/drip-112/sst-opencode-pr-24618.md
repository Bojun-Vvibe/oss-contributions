# sst/opencode PR #24618 — guard against undefined MCP tool output causing `output.split` crash

- **PR**: https://github.com/sst/opencode/pull/24618
- **Author**: @leandrosnx
- **Head SHA**: `c11a6d474545129fff4cd312f1299cc311c53a6b`
- **Size**: +3 / −1
- **Files**: `packages/opencode/src/session/message-v2.ts`, `packages/opencode/src/tool/truncate.ts`

## Summary

Two-line fix for issue #24402: an MCP tool that returns only image content (or an empty content array) ends up with `part.state.output === undefined`, and any downstream consumer that calls `.split(...)` on it (the truncation pipeline) throws `TypeError: undefined is not an object (evaluating 'output.split')`. The fix is defensive nullish coalescing at the two call sites that consume `state.output` plus an early-return in the lower-level truncate primitive.

## Verdict: `merge-after-nits`

Correct, minimal fix. The real shape of the issue is upstream — `part.state.output` should be normalized to `""` at the point where the MCP tool result is materialized, not patched at every consumer — but this PR is the right shape for a stable-branch hotfix. Add a regression test before merging.

## Specific references

- `packages/opencode/src/session/message-v2.ts:854` — the original line called `truncateToolOutput(part.state.output, options?.toolOutputMaxChars)` directly. With image-only MCP responses, `part.state.output` is `undefined`, and inside `truncateToolOutput` the `text.length <= maxChars` short-circuit was *not* reached because the function happily passed `undefined` through to `text.slice(...)` later. The new call site coalesces to `""` via `part.state.output ?? ""`.
- `packages/opencode/src/session/message-v2.ts:321` — the inner guard `if (!text) return ""` at the top of `truncateToolOutput` is a belt-and-suspenders fix. It catches both `undefined` and the empty string. This is fine, but it makes the `?? ""` at the call site at line 854 strictly redundant — the function now tolerates `undefined`. I'd keep one or the other for clarity, not both. (Recommendation: keep the inner guard, drop the `?? ""` at the call site so the type system surfaces the `string | undefined` contract.)
- `packages/opencode/src/tool/truncate.ts:87` — adds `if (text == null) return { content: "", truncated: false } as const` to the public `Truncate.output` Effect. Good — this is the right layer to pin the contract because it's the tool-facing entrypoint. Note the use of `== null` (loose) which catches both `null` and `undefined`; consistent with the rest of the codebase.

## Nits

1. No test was added. A regression test for issue #24402 is essential — otherwise the same crash will reappear the next time someone refactors `truncateToolOutput`. Construct an MCP tool result with `content: [{ type: "image", ... }]` (no text part) and assert `toModelMessagesEffect` produces a message without throwing, with `outputText === ""`.
2. The deeper question is *why* `part.state.output` is `undefined` in the first place. The MCP tool-result materialization path should normalize to `""` for image-only or empty results. That's a separate PR, but worth filing as a follow-up so the workaround at line 854 can eventually be removed.
3. The `as const` on `{ content: "", truncated: false } as const` (line 87) is fine but slightly inconsistent with the neighboring return types in the same Effect — they're plain objects. Minor.

## What I learned

`output.split` crashes from undefined inputs are a recurring smell in tool-result pipelines because tool outputs are union-typed at the protocol layer (text | image | resource | nothing) but consumers assume "string-or-empty-string". The right fix usually lives at the materialization boundary, not the consumer — but a two-line consumer guard is a legitimate hotfix when you don't want to risk a behavior change in the materialization layer. The pattern of `if (!text) return ""` at the top of a string-shaping helper is also a useful trick: it lets the type signature stay `(text: string)` while the implementation tolerates `(string | undefined)` accidentally being passed in.
