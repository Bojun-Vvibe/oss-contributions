# sst/opencode#24943 — feat: use small models for explore subagents

- PR: https://github.com/sst/opencode/pull/24943
- Head SHA: `ed9621271282cb22e424a563f1d5fd6ce96c31ca`
- Author: nexxeln (Shoubhit Dash)
- Base: `dev`
- Size: +138 / -21 across 4 files

## Summary

Refactors the inline small-model priority list at
`provider/provider.ts:1614-1629` (which was hardcoded with three special
cases for `opencode`, `github-copilot`, and "everyone else") into a typed
`Record<string, string[]>` map at the new `provider/small-model.ts`. Then
wires `TaskTool` to use `provider.getSmallModel(msg.info.providerID)` *only*
when spawning the built-in `explore` agent without an explicit model
override (`tool/task.ts:104`). Falls back to the parent agent model when no
small model is available for the provider.

## Notable design choices

- `provider.ts:1617` — the new lookup `priority[providerID] ?? []` cleanly
  replaces the previous if/else cascade. Bedrock keeps its cross-region
  prefix special case at `provider.ts:1620+` because model IDs there have
  region prefixes (`global.`, `us.`, `eu.`) that need substring matching,
  not exact equality. Good preservation of existing behavior.
- `task.ts:104` — the gate is `!next.model && next.name === "explore"`,
  which is the right scope: explicit user model selection wins, and only
  the *explore* agent (which is read-only and can tolerate weaker reasoning)
  gets downgraded. Other built-in agents (general, build, plan) keep parent
  model.
- `small-model.ts:7-9` — `opencode` provider priority is `gpt-5.4-mini →
  gemini-3-flash → claude-haiku-4-5`. This is a behavior change from the
  prior list which had `gpt-5-nano` as the only opencode entry. Noted but
  probably intentional (5-nano was too weak for explore even on opencode's
  bundled provider).

## Concerns

1. **Loss of fallthrough at `provider.ts:1617`.** Before: a model not
   matching any priority entry fell through to `return undefined` and the
   caller used the parent model. After: same behavior, but the tight
   `priority[providerID] ?? []` means that *any* provider not in the map
   (custom providers, plugin-installed providers, `lmstudio`, `ollama`,
   `together`, `groq`, `cerebras`, `deepseek`, `fireworks`, `mistral`, etc.)
   silently gets `[]` and skips the small-model path entirely. Add a
   comment at `small-model.ts:3` explicitly stating that providers omitted
   from this map intentionally fall back to parent-model behavior, so the
   next contributor doesn't add a `console.warn` here.
2. **No test for the negative path.** `task.test.ts` adds 56 lines but the
   diff doesn't show whether it covers `(name !== "explore", model unset)`
   asserting the parent model is used, nor `(name === "explore", model
   set)` asserting the explicit model wins. Both are easy to break in a
   future refactor that drops the conjunction at `task.ts:104` to a single
   condition.
3. **Map key `"opencode-go"` at `small-model.ts:11`.** Is this a real
   provider ID, or aspirational? If aspirational (the provider doesn't
   exist yet), drop it — dead config drifts. If it's gated behind a
   feature flag, add a comment.
4. **`xai` priority at `small-model.ts:50-58`.** Eight entries, several of
   which are dated/version-suffixed (`grok-4.20-0309-non-reasoning`,
   `grok-3-mini-fast-latest`). This list will rot fast as xAI ships. Worth
   considering a `getSmallModel()`-side glob or "newest-version-wins" so
   the list doesn't need updating per release.
5. **`amazon-bedrock` at `small-model.ts:60-69`** mixes `anthropic.*`,
   `openai.*`, `mistral.*`, `amazon.*`, `meta.*` namespaces. The Bedrock
   special case at `provider.ts:1620+` does substring matching, so e.g.
   `anthropic.claude-haiku-4-5` will also match `global.anthropic.claude-
   haiku-4-5`. Confirm this still works after the refactor (the early
   `if (provider.models[item]) return ...` short-circuit at line 1618 was
   added; if a Bedrock user happens to have a model literally named
   `anthropic.claude-haiku-4-5` *and* a region-prefixed variant, the
   short-circuit wins and skips the cross-region preference logic).

## Verdict

**merge-after-nits** — clean refactor that meaningfully reduces the cost of
the explore agent (which fires often and has bounded read-only scope, so
weaker models are fine). The test gap on the negative path (item 2) and the
Bedrock short-circuit interaction (item 5) are the only items I'd want
verified before merge.
