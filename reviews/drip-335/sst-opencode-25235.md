# Review: sst/opencode #25235

- **Title:** feat: add visual feedback for valid skill slash commands
- **Head SHA:** `2353da6616eecd61844b5b28b986554746a89887`
- **Size:** +265 / -15
- **Files touched:**
  - `packages/app/src/components/prompt-input.tsx`
  - `packages/app/src/components/prompt-input/editor-dom.test.ts`
  - `packages/app/src/components/prompt-input/editor-dom.ts`
  - `packages/core/src/util/skill-token.test.ts`
  - `packages/core/src/util/skill-token.ts`
  - `packages/opencode/src/cli/cmd/tui/component/prompt/index.tsx`
  - `packages/opencode/src/cli/cmd/tui/context/theme.tsx`

## Critique

The change extracts skill-token recognition into a shared `@opencode-ai/core/util/skill-token` module and threads it through both the web prompt-input and the TUI prompt component, decorating the leading skill command with a styled `data-type=skill` element. The split between `leadingSkillCommandToken` (pure helper) and the renderer-specific decoration code is clean.

Concerns to verify before merge:

- `prompt-input.tsx:537` — `skillTokenForParts` short-circuits on `store.mode !== "normal"`. This is correct intent (don't decorate in shell/agent mode) but a snapshot test for mode-switching while the editor already contains a skill token would protect against stale decoration when mode flips mid-session. Currently `skillTokenDecorationMatches` will detect the mismatch, but a regression test makes the contract explicit.
- `prompt-input.tsx:728-730` — the predicate for "is this a normalized editor child?" was tightened from string compare on `dataset.type` to `isPillElement(el) || isSkillTokenElement(el)`. Good factoring, but make sure both helpers in `editor-dom.ts` also handle the case where `dataset` is missing on a stray text-wrapped span (paste from external source). A defensive `el.dataset?.type` check would mirror the rest of the file (`prompt-input.tsx:546`).
- `prompt-input.tsx:823-829` — the new effect signature is `() => [prompt.current(), syncedCommands(), store.mode] as const`. This will refire `reconcile` every time commands sync (network), which can race with active typing. Confirm `composing()` still gates IME edits and add a brief debounce or equality check on the commands list — otherwise users on flaky networks may see cursor jumps.
- `appendTextWithSkillToken` (line 736-752) — the math is correct for a single token at the head, but the helper is named generically; if/when multiple skill tokens are supported it will silently render only the first. A `// only-handles-leading-token` comment or a TODO would prevent future surprise.
- The new theme entry `text-text-interactive-base` at `prompt-input.tsx:1413` should also be checked against the dark/high-contrast themes — bold + interactive blue can clip contrast in some themes.

No banned strings, no security concerns, tests cover the new helper. Mostly polish.

## Verdict

`merge-after-nits`
