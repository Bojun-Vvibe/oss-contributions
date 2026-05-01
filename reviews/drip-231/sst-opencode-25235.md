# sst/opencode#25235 — feat: add visual feedback for valid skill slash commands

- **Author:** gpaiva00
- **Head SHA:** `76747f37dc994c5f6f91745395dc5877986db6e5`
- **Base:** `dev`
- **Size:** +265 / -15 across 7 files
- **Files changed:** `packages/app/src/components/prompt-input.tsx`, `prompt-input/editor-dom.{ts,test.ts}`, `packages/core/src/util/skill-token.{ts,test.ts}`, `packages/opencode/src/cli/cmd/tui/component/prompt/index.tsx`, `packages/opencode/src/cli/cmd/tui/context/theme.tsx`

## Summary

Adds inline visual feedback in the prompt input when the user types a recognized `/skill-name` token at the start of a message: the token gets highlighted (web app) or styled (TUI) so the user knows the slash command was actually recognized as a skill rather than a generic provider/MCP/custom command. Introduces a small `leadingSkillCommandToken` core utility that classifies the leading token by source.

## Specific code references

- `packages/core/src/util/skill-token.ts:14-37`: `leadingSkillCommandToken(input, commands)` is the source-of-truth classifier. The regex `/^\/(\S+)/` matches the leading non-whitespace word after `/`, then filters the candidate list by exact name and crucially **rejects matches if any candidate with that name has `source !== "skill"`**: `if (matches.some((c) => c.source !== "skill")) return`. This is the load-bearing predicate — a custom command shadowing a skill name correctly suppresses the skill highlight rather than incorrectly stamping it (test pinned at `skill-token.test.ts:48-53`).
- `skill-token.test.ts:18-32`: pins the leading-only behavior with `/prompt do this` and `/prompt\ndo this` both producing token `/prompt` — the regex `\S+` correctly stops at any whitespace including newlines, which is the right behavior for a chat-style multi-line input. The partial/unknown arm at `:36-39` also correctly rejects `/pro` (prefix of `/prompt`) so the highlight only fires on exact matches.
- `editor-dom.ts:3-8,194-199`: new `createSkillTokenElement(content)` returns a `<span data-type="skill">` carrying the raw text. Symmetric with the existing `data-type="file"` and `data-type="agent"` pill convention, so the `:host {[data-type=skill]}` CSS hook at `prompt-input.tsx:1413` (`text-text-interactive-base font-bold`) cleanly themes the new node without disrupting the existing pill styling.
- `editor-dom.ts:212-229`: new `setRangeWithinText(range, node, offset, edge)` recurses into the skill span's text descendant so `setCursorPosition` and `setRangeEdge` can land the caret *inside* the token (a user typing past `/prompt` should keep the cursor at the right text offset). Important contrast with the pill behavior at `:284-287`, which uses `setStartBefore`/`setStartAfter` because pills are atomic — skill tokens are editable text wrapped for display, not atomic objects, and this distinction is preserved correctly.
- `prompt-input.tsx:35-50`: new `skillTokenForParts` and `skillTokenDecorationMatches` predicates. The latter at `:46-50` returns false when the editor's first child is a `data-type=skill` span but `textContent !== token.token`, so the reconciler at `:113,121` correctly re-renders when the user edits the skill name itself (e.g. typing one more character after `/prompt`). Without this predicate the existing `isPromptEqual` short-circuit would skip the re-render and the highlight would go stale.
- `prompt-input.tsx:73-88`: `appendTextWithSkillToken` slices the text into `before`/`middle`/`after` around the token boundaries, only emitting a skill-token element for the slice that overlaps `[token.start, token.end)`. The bounds-check `token.end <= offset || token.start >= offset + content.length` correctly handles the case where the token straddles a part boundary (it can't, in practice, because the token must be at offset 0 and parts are append-order — but the explicit bounds check makes the contract robust).
- `prompt-input.tsx:128-133`: the reactive `on(...)` dependency is correctly widened from `prompt.current()` to the tuple `[prompt.current(), syncedCommands(), store.mode]` so the editor re-reconciles when the available commands list changes (e.g. a skill is added at runtime) or the user toggles to shell mode (where the highlight should not appear, gated by `store.mode !== "normal"` at `:36`).

## Verdict

**merge-after-nits**

Solid, well-factored change with a real test surface and the load-bearing same-name-collision predicate explicitly tested. Nits: (a) `appendTextWithSkillToken` at `prompt-input.tsx:73` doesn't handle the case where `parts` later contain a non-text part (file/agent pill) before the skill token's end-offset would be reached — in practice the skill token is always at offset 0 in the first text part, but a load-bearing comment naming that assumption would harden the function; (b) the `getNodeLength` walk that `setRangeWithinText` depends on at `editor-dom.ts:223` should arguably be exported alongside the new helper since the recursive contract now spans the whole module; (c) the `[data-type=skill]:font-bold` styling at `prompt-input.tsx:1413` is hardcoded — consider whether bold is the right choice across all themes or whether it should pull from the theme tokens like the existing `data-type=file` styling does.
