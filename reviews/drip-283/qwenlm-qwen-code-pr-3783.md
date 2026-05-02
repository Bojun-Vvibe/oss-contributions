# QwenLM/qwen-code PR #3783 — feat(cli): Add ability to switch models non-interactively from the cli

- Head SHA: `cf50ca08d138366f1465240905b4e3ce9df97b60`
- Size: +59 / -5, 2 files

## Specific refs

- `packages/cli/src/ui/commands/modelCommand.ts:17-27` — new `getAvailableModelIds(context)` helper pulls IDs from `config.getAvailableModels()` for completion. Safe nil guard on missing config.
- `packages/cli/src/ui/commands/modelCommand.ts:46-54` — completion handler now returns `getAvailableModelIds(_context).filter(id => id.startsWith(partialArg))` for non-`--fast` partial args. Previously returned `null` (no completion) when the user started typing a model name.
- `packages/cli/src/ui/commands/modelCommand.ts:131-167` — new interactive-mode branch: if `args !== ''`, parse first whitespace-delimited token as model name, persist via `settings.setValue(getPersistScopeForModelSelection(...), 'model.name', modelName)` and call `await config.setModel(modelName)`. Returns one of two info messages depending on whether the model is in the registry — graceful degradation when switching to upstream models not in config.
- `packages/cli/src/ui/commands/modelCommand.ts:169-173` — non-interactive branch tightened: `args.trim().split(' ')[0]` instead of `args.trim()`. Means trailing whitespace or extra args no longer pass through verbatim.

## Assessment

Useful UX feature with a clear analog (the existing `--model` launch arg). Implementation is small and contained to one command file. The "first token wins" parsing rule (PR explicitly notes it) is a behavior change for invalid `/model` syntax — extraneous args used to be ignored, now the first one is interpreted as a model name. This is called out as a tradeoff in the PR's "Scope / Risk" section, which is the right thing to do.

Concerns to address before merge:
- The interactive branch (lines 131-167) duplicates the settings-write + `setModel` logic that already exists in the non-interactive branch below it. Worth extracting a shared `applyModelSwitch(settings, config, modelName)` helper to avoid drift.
- No test added for the new interactive behavior — `modelCommand.test.ts` only changes the description-string assertion. Author tested via screenshots, but a unit test pinning "args=`gpt-X`, executionMode=`interactive` → settings.setValue called with `model.name`/`gpt-X`" would be cheap.
- `args.trim().split(' ')[0]` will incorrectly handle model IDs containing spaces (rare but possible in custom configs). The "no spaces in model names" assumption is mostly safe but should be a comment.
- Description string ends with a trailing space (`'... immediately). '`) — minor cosmetic, drop it.

verdict: merge-after-nits
