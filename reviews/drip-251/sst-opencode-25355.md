# sst/opencode#25355 — fix(tui): bind home/end to line start/end in input

- **PR:** https://github.com/sst/opencode/pull/25355
- **Head SHA:** `d3bda7e85b1b41c05e6d6e9f9754e7ccf4b76f17`
- **Stats:** +192 / -192 across 20 files (1 source `keybinds.ts`, 1 tips view, 18 locale `keybinds.mdx`)
- **Closes:** #14899 (supersedes #19135 because keybind defaults moved out of `config.ts` into `keybinds.ts`)

## Context
Default keybinds claimed bare `home` / `end` for `input_buffer_home` / `input_buffer_end` *and* assigned `ctrl+g,home` / `ctrl+alt+g,end` to `messages_first` / `messages_last`. Result: pressing Home/End (or fn+Left/fn+Right on macOS) inside a multiline prompt scrolled message history instead of doing what every readline-derived terminal app does — jump to start/end of the current line. Issue #14899 is exactly this report.

## Design
Single source-of-truth diff at `packages/opencode/src/config/keybinds.ts:46-90` swaps three buckets in lockstep:

| Action | Before | After |
|---|---|---|
| `messages_first` / `_last` | `ctrl+g,home` / `ctrl+alt+g,end` | `ctrl+g` / `ctrl+alt+g` |
| `input_line_home` / `_end` | `ctrl+a` / `ctrl+e` | `ctrl+a,home` / `ctrl+e,end` |
| `input_select_line_home` / `_end` | `ctrl+shift+a` / `ctrl+shift+e` | `ctrl+shift+a,shift+home` / `ctrl+shift+e,shift+end` |
| `input_buffer_home` / `_end` | `home` / `end` | `ctrl+home` / `ctrl+end` |
| `input_select_buffer_home` / `_end` | `shift+home` / `shift+end` | `ctrl+shift+home` / `ctrl+shift+end` |

Buffer-jump moves to `ctrl+home` / `ctrl+end`, the long-standing convention in VS Code, JetBrains, Sublime — which frees the bare Home/End slot for line-start/line-end. The three docs strings in `tips-view.tsx:78-79` also drop the `or {Home}` / `or {End}` aliases that no longer apply. The 18 locale `keybinds.mdx` files are mechanical mirrors of the schema defaults, so the JSON snippets in each match the new shape. Override semantics are preserved because defaults only apply when a key is absent in `tui.json`.

## Risks
- **Muscle memory regression** for users who learned the previous `home`/`end` = buffer-jump shape. They get the universal convention back, which is a *trade*, not a pure win — author acknowledges this.
- **Multi-binding shape (`"ctrl+a,home"`)** assumes the keybind parser treats commas as alternatives. Worth one-line confirmation that `ctrl+e,end` doesn't get parsed as a chord.
- **No CHANGELOG entry** — this is a default-flip touching a load-bearing UX surface; one bullet under "TUI" would help users find it when their muscle memory breaks.
- **Localization debt is constant** — 18 files updated symmetrically (good), but if any locale has hand-edited prose around the JSON block describing what `home` does, it's now stale.

## Suggestions
1. CHANGELOG note flagging the default-flip with a one-line migration: "If you relied on bare `home`/`end` for buffer jump, restore via `tui.json` override."
2. Confirm-via-comment that the keybind parser splits on `,` for alternatives (the existing `messages_first: "ctrl+g,home"` example proves it does, but the new `ctrl+a,home` shape conflates the chord-vs-alt question more than the old shape did).
3. Grep the 18 locale `.mdx` files for any prose mentioning `Home` / `End` / `главная` / `主页` etc. that explains the old behavior — table changed but surrounding paragraphs may not have.

## Verdict
**merge-after-nits** — correct fix, correct breadth (source + tips + 18 locales + supersession of #19135 documented), but a one-line CHANGELOG and a parser-comment confirmation would close the loop.

## What I learned
The "the default has *two* meanings at once" bug — when bare `home` is bound to `input_buffer_home` *and* listed as alt for `messages_first`, the conflict resolution silently routes one of them to the wrong handler depending on focus context. The clean fix isn't to add precedence rules; it's to make the *defaults* unambiguous (one slot per key) and let users with the old habit opt back in via override. The supersession of #19135 (older PR pointed at since-renamed file) is the right move — note it in the PR body, don't just close the older one with no context.
