# Review: sst/opencode#25140 — Support multiple Zed selections in TUI context

- PR: https://github.com/sst/opencode/pull/25140
- Head SHA: `a05ac268bc6e836c85e41c2487eab8481a91e1e3`
- Author: kitlangton
- Files: ~6 changed (+278/-89)
- Merged: 2026-04-30

## Stated purpose
The Zed adapter only ever forwarded a *single* selection range into the TUI editor-context model, so users with multi-cursor selections in Zed lost all but one range when reaching for the "send context with prompt" workflow. This PR generalizes the editor-context shape to `EditorSelection.ranges: Range[]`, surfaces multiple ranges in the prompt footer chip, keeps the legacy single-range websocket payload backward-compatible, and adds focused tests.

## What actually changed
1. **`prompt/index.tsx:84-114`** — `getEditorSelectionKey(selection: EditorSelection)` (a single-range key builder) is replaced by three helpers: `hasEditorRangeSelection(range)` predicate (true iff start/end differ), `getEditorRangeLabel(range)` returning `#42` for single-line or `#42-58` for multi-line, and `formatEditorContext(selection)` that builds the system-reminder string with `Selection 1: #L-L`, `Selection 2: ...` prefixes when `selected.length > 1`. The single-selection path is preserved (no `Selection N:` prefix when count is 1) — important for prompt-stability so users don't see a wording change when their workflow didn't change.
2. **`prompt/index.tsx:139-156`** — `editorPath` / `editorSelectionLabel` memoizers now derive from a new `editorContext()` memo that filters out a "dismissed" selection key (tracked via `dismissedEditorSelectionKey` signal). `editorSelectionLabel` builds a `#L-L +N` style label where `+N` reports additional-range count, e.g. `#10-15 +2` for three ranges total — concise, scannable, matches the "show one, hint the rest" pattern used for collapsed lists elsewhere in the TUI.
3. **`@tui/context/editor`** — exports `editorSelectionKey` (now stable across the whole `EditorSelection`, not just one range) so the dismissal logic can deduplicate multi-range selections as a single unit.
4. **Backward compatibility** — legacy single-selection websocket payloads continue to deserialize into `{ ranges: [...] }` shape with one element, per the PR summary.
5. **Tests** — `editor-context-zed.test.ts`, `editor-context.test.tsx` exercise multi-range Zed payloads and the new label/format paths.

## Quality / risk observation
The `formatEditorContext` shape — emit the file-opened reminder when `selected.length === 0` and emit one `Note: ...` block per range with `Selection N:` prefixes only when count > 1 — is the right empathy pattern for prompt engineering: don't add ceremony for the common case (one range) and don't suppress structure for the multi-range case where the model genuinely needs to disambiguate. One minor nit: the `system-reminder` block in the multi-range path joins `ranges` with `\n` but each range's template already ends in `\n\n`, which produces `\n\n\n` between ranges and a `\n\n\n This may or may not be relevant to the current task.</system-reminder>` tail with three blank lines before the "may or may not" sentence. That's harmless to the model but makes the rendered prompt look ragged when echoed back in debug logs — a `.join("")` (with the trailing `\n\n` already in the per-range template) would be tighter. Second nit: `editorSelectionLabel` returns `[label, additionalCount].filter(Boolean).join(" ")` — if `getEditorRangeLabel` returns `undefined` (zero-width "selection"), the `+N` count would render alone like `+2`, which is confusing UX. The earlier `.find(hasEditorRangeSelection) ?? ranges[0]` fallback partially guards against this but doesn't eliminate it. Both nits are cosmetic. The dismissal-key reset story (what re-enables a dismissed selection?) is implicit — if Zed sends a *different* selection set, the key changes and the new selection is visible; if the user reselects the *same* range, it stays dismissed until something else happens. That's defensible but worth a one-line comment. Zero correctness risk.

## Verdict
`merge-after-nits`
