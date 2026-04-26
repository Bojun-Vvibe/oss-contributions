---
pr: 24422
repo: sst/opencode
sha: 8e6c799cd808710f379729056e036d11d679146b
verdict: merge-after-nits
date: 2026-04-26
---

# sst/opencode #24422 — feat(session): persist invoking slash command on user messages

- **Author**: rustyaos
- **Head SHA**: 8e6c799cd808710f379729056e036d11d679146b
- **Link**: https://github.com/sst/opencode/pull/24422
- **Size**: ~50 diff lines across `packages/opencode/src/session/message-v2.ts` and `packages/opencode/src/session/prompt.ts`.

## Scope

Adds an optional `command: { name, arguments, source: "command"|"mcp"|"skill" }` field to the v2 `User` message schema and threads it through `PromptInput` so that when a user invokes a message via a slash-command path, the originating command is persisted on the resulting `User` message rather than getting flattened into a plain text turn. This unblocks downstream UI / replay / analytics that wants to know "this turn started life as `/plan`".

## Specific findings

- `packages/opencode/src/session/message-v2.ts:391-400` — clean addition of an optional `command` struct with a `Schema.Literals(["command", "mcp", "skill"])` source enum. Schema-only, no migration needed since it's `Schema.optional`. Fine.
- `packages/opencode/src/session/prompt.ts:1655-1660` — when constructing a User message inside the slash-command branch, populates `command: { name: input.command, arguments: input.arguments, source: cmd.source ?? "command" }`. Reasonable default.
- `packages/opencode/src/session/prompt.ts:956-959` — the conditional spread `...(input.command ? { command: input.command } : {})` is asymmetric with how other optionals on this struct are passed (most just rely on `Schema.optional` filtering undefined). Either pattern works but the bare `command: input.command` would round-trip the same. Minor style nit.
- `packages/opencode/src/session/prompt.ts:1730-1736` — `PromptInput` schema gets the same nested struct. Note this is now declared **twice** (once on `User`, once on `PromptInput`) with identical shape — extracting a shared `CommandRef = Schema.Struct({...})` would prevent drift if a fourth source type is ever added.
- No tests added. The shape is small enough that schema validation will catch most regressions, but a single round-trip test through `prompt.ts:1655` showing the `command` field survives onto the persisted `User` would protect against the next refactor that drops it again.

## Risk

Low. Optional field, additive schema change, no storage migration. The only correctness risk is the duplicated schema definition drifting; the only UX risk is downstream consumers ignoring the new field, which is the safe failure mode.

## Verdict

**merge-after-nits** — extract the shared `CommandRef` schema (or accept the duplication explicitly with a comment), and add one assertion that `User.command` survives the prompt → user-message materialization.
