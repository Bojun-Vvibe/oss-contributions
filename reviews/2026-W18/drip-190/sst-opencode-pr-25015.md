# sst/opencode #25015 — fix: make Home/End move caret to line edges in prompt input

- **URL:** https://github.com/sst/opencode/pull/25015
- **Head SHA:** `da9e2cf498d74594b12d247631220cf2fea94f1b`
- **Files:** `packages/app/src/components/prompt-input.tsx` (+15/-0)
- **Verdict:** `merge-after-nits`

## What changed

The prompt-input is a `contenteditable` element with mixed inline content (text nodes + `<br>` + non-editable "pill" widgets for file mentions / commands). Browser default Home/End navigation across such mixed content is unreliable: caret can land mid-pill or skip lines depending on which DOM node the keypress is dispatched to. `prompt-input.tsx:1167-1180` adds a manual handler:

- On `Home` / `End` (without Meta/Alt — those reach the OS-level word/document shortcuts on macOS):
  - `direction = key === "Home" ? "backward" : "forward"`
  - `granularity = ctrlKey ? "documentboundary" : "lineboundary"`
  - `alter = shiftKey ? "extend" : "move"`
  - `selection.modify(alter, direction, granularity); event.preventDefault()`
- Returns early so the rest of the keydown handler (Shift+Enter, IME, etc.) is short-circuited.

## Why it's right

- `Selection.modify(alter, direction, granularity)` is exactly the right API for this — it routes through the browser's text-editing primitives (which understand line-boundary across mixed inline content) instead of relying on default contenteditable bubbling.
- The `alter` mapping is correct: `"move"` collapses the selection to the new caret position; `"extend"` preserves the anchor and extends to the new focus, which is the right Shift+Home/End behavior.
- The `granularity` mapping respects the OS convention: Ctrl+Home/End → start/end of document, plain Home/End → start/end of line.
- The `metaKey || altKey` early-return is correct on macOS: Meta+Left/Right is the macOS line-edge shortcut and is handled by Cocoa's text input system; intercepting it would double-handle. Alt+Home/End is unbound on most platforms but worth the same skip.
- The placement *before* the Shift+Enter / IME branches at `:1182` is correct — Home/End are never IME composition keys, so short-circuiting before the IME logic is safe.

## Nits

- **`window.getSelection()` returns `null` on detached documents.** The `if (!selection) return` guard is correct but doesn't `event.preventDefault()` before returning, so on a null-selection browser the default Home/End would fire (probably fine, but inconsistent with the manual path that *does* preventDefault). Cheaper to `event.preventDefault()` unconditionally at the top of the block.
- **No coverage of `selection.modify` failure modes.** Some browsers (older WebKit, certain mobile contexts) silently no-op on `selection.modify` for granularities they don't support. A `try { selection.modify(...) } catch { /* fall through to default */ }` with the preventDefault inside the try would degrade gracefully on those engines instead of leaving the caret unchanged.
- **Granularity choice for `"documentboundary"` is correct on Chromium/Firefox** but `"documentboundary"` is technically a non-standard extension to the WebKit `Selection.modify` spec. If a Safari-targeting smoke test exists, exercise `Ctrl+Home` / `Ctrl+End` there explicitly. (On macOS Safari, Ctrl+Home/End are fairly uncommon — users hit Cmd+Up/Down — so the practical impact is small.)
- **No regression test in this diff.** A jsdom unit test on `PromptInput` that synthesizes a `KeyboardEvent("keydown", { key: "Home" })` and asserts `selection.modify` was called with `("move", "backward", "lineboundary")` would lock the contract. The argument-mapping table at `:1173-1175` is the kind of thing that silently inverts in a future refactor (e.g. someone "simplifies" by reordering args).
- **Behavior on macOS without modifiers.** On macOS, the platform convention for "go to line start" is *Cmd*+Left, not Home — many Mac keyboards don't even have a Home key. This handler is correct when Home *is* pressed (e.g. external keyboard) but doesn't address the missing platform-affordance. Out of scope for this fix; note for the follow-up.

## Risk

Low. Tightly scoped to two specific keys, gated by modifier-key checks, with `preventDefault` confining the effect. Worst case if `Selection.modify` misbehaves on an exotic browser is "Home/End does nothing" rather than a worse default; in that case users will report it and the granularity fallback can be added.
