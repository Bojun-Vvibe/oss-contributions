---
pr: 24317
repo: sst/opencode
sha: 94a1f2a07faf6f2168c9807e674a51bd58da86ff
verdict: merge-after-nits
date: 2026-04-27
---

# sst/opencode #24317 — feat(tui): assign model to specific agent via Ctrl+E

- **Head SHA**: `94a1f2a07faf6f2168c9807e674a51bd58da86ff`
- **URL**: https://github.com/sst/opencode/pull/24317
- **Size**: medium (212 additions / 11 deletions across 7 files)

## Summary
Adds a Ctrl+E keybind in the agent dialog that opens the model selector pre-targeted at the highlighted agent, and persists the chosen `provider/model` back into `opencode.json`/`opencode.jsonc` under `agent.<name>.model`. Also bundles a second feature: a new `vision_model` config field plus `Provider.getVisionModel()` fallback that scans all providers for an image-capable model when the active model can't accept image input.

## Specific findings
- `packages/opencode/src/cli/cmd/tui/component/dialog-agent.tsx:23-31` — `allVisible()` filter excludes `["explore", "general", "translator"]` from the picker. This list is hardcoded inside the local context; if any of those are renamed in the agent registry the filter will silently stop working. Worth either reading the exclusion set from agent metadata (`item.hidden`?) or a TODO comment.
- `dialog-agent.tsx:42-46` — the `onSelect` guard now silently no-ops when the user picks a `subagent`. There is no toast/feedback explaining why nothing happened. A `toast.show({ message: "subagents cannot be set as primary", variant: "warning" })` would prevent the "is the dialog frozen?" experience.
- `packages/opencode/src/cli/cmd/tui/context/local.tsx:148-260` — `setForAgent()` does the right thing (jsonc-aware edit via `modify()` + `applyEdits()`), but the `isJsonc` heuristic at the line `const isJsonc = file !== "{}" ? file.includes("//") || file.includes("/*") : false` is fragile: a string value like `"description": "see http://x"` would falsely trigger `isJsonc`. Cheap fix: prefer the path that actually exists on disk (you already `try { Bun.file(configPath).text() } catch { try configPathC }` above — keep the same precedence).
- `packages/opencode/src/provider/provider.ts:1647-1672` — `getVisionModel()` iterates `s.providers` in object-key order. JS object iteration order is insertion-order but is not the user's "preferred provider order". If a user has 6 providers configured the first vision-capable one wins, not the cheapest. The comment "Do NOT prefer the same provider — if the user has run out of credits …" is good intent but the chosen alternative (object iteration) is essentially random. Consider documenting that and/or letting `vision_model` config win, which it already does at `:1652-1655`.
- `packages/opencode/src/provider/provider.ts:1706` — `Service.of({...})` updated to include `getVisionModel`; this is the right plumbing, but the new public API isn't exercised by a test in this PR. The `Provider` service has companion tests elsewhere; one unit covering "current model has no image capability → returns first vision-capable model" would catch regressions.
- The PR is **two features in one** (per-agent model assignment + vision-model fallback). The vision-model code is fine on its own but doesn't belong with a Ctrl+E TUI keybind change. Splitting would make the changelog cleaner and reviews faster.

## Risk
Low–medium. The jsonc-edit path is the riskiest piece: a stray `//` inside a description string would route writes to a `.jsonc` file the user never had. Worst case is a duplicate config file appearing on disk; no data loss, but confusing.

## Verdict
**merge-after-nits** — split the vision-model feature out, add a no-op-feedback toast for the subagent rejection path, replace the `isJsonc` heuristic with a "which file did I successfully read?" check, and add a single `getVisionModel` unit test.
