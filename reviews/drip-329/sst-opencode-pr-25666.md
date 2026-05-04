# sst/opencode #25666 — feat(tui): add input.intercept API for plugin keydown interception

- SHA: `05ff633147d7ba2dd3bc87266e1d08777a49c884`
- State: OPEN, +42/-1 across 6 files
- Files: `packages/opencode/src/cli/cmd/tui/component/prompt/index.tsx`, `packages/opencode/src/cli/cmd/tui/component/prompt/intercept.ts` (new), `packages/opencode/src/cli/cmd/tui/plugin/api.tsx`, `packages/opencode/src/cli/cmd/tui/plugin/runtime.ts`, `packages/opencode/test/fixture/tui-plugin.ts`, `packages/plugin/src/tui.ts`
- Closes: #1764

## Summary

Adds an `input.intercept` plugin API that lets plugins consume prompt-textarea keystrokes before default handling. Module-level handler list, registration order, first-truthy wins. Foundation for vim-mode / macro / custom-keybinding plugins without core changes.

## Notes

- `packages/opencode/src/cli/cmd/tui/component/prompt/intercept.ts:1-21` — new module. Uses a module-level `const handlers: InputInterceptHandler[] = []`. The `register` returns an `Unregister` closure that splices via `indexOf`. Correct, but the registry is process-global — across hot-reload or multiple `Prompt` mounts, unregister responsibility is entirely on the caller. Worth a one-liner doc comment that handlers persist for the lifetime of the TUI process unless the returned unregister is invoked.
- `packages/opencode/src/cli/cmd/tui/component/prompt/index.tsx:1108-1113` — dispatch is wired right after the `useTextareaKeybindings` early-return and *before* the Ctrl+V clipboard-image branch. That ordering is intentional and correct: a vim-mode plugin needs to see Ctrl+V before paste-image kicks in. Worth calling out in the PR description because it is a subtle ordering choice.
- `packages/opencode/src/cli/cmd/tui/component/prompt/index.tsx:1111` — `e.preventDefault()` runs after dispatch returns truthy, mirroring the pattern used by the `useTextareaKeybindings` block immediately above. Consistent.
- `packages/opencode/src/cli/cmd/tui/plugin/api.tsx:319-323` — `input.intercept` exposed via `PromptIntercept.register`. Single-method object leaves room for future additions (`input.before`, `input.after`) without an API break.
- `packages/opencode/src/cli/cmd/tui/plugin/runtime.ts:538` — `input: api.input` plumbed into the per-plugin scoped api. No isolation between plugins (handlers from plugin A still see keys destined for plugin B's textarea); acceptable for v1 but worth documenting.
- `packages/plugin/src/tui.ts:18,449,479-481` — public type `TuiInputInterceptHandler = (evt: ParsedKey, input: TextareaRenderable) => boolean | void` exported on the `TuiPluginApi` surface. The `boolean | void` return matches the dispatcher's `if (handler(evt, input)) return true` truthy check — `void` falls through, `true` consumes. Clean.
- `packages/opencode/test/fixture/tui-plugin.ts:175-177` — fixture stub `intercept: () => () => {}` keeps existing 20 plugin tests green without per-test wiring. Good.
- Nit: no unit test exercises actual interception ordering / consumption. PR body claims existing 20 tests still pass, but a 1-test addition that registers two handlers and verifies first-truthy-wins would lock the contract. Not a merge blocker given the surface area is ~21 LOC.
- Nit: `intercept.ts` ends without a trailing newline (`\ No newline at end of file` in the diff). Repo style elsewhere uses trailing newlines.

## Verdict

`merge-after-nits` — clean, minimal API; correct ordering. Add a short doc comment on global-registry lifetime, fix the trailing newline, and ideally land one ordering-contract test before merge.
