# sst/opencode #25355 — fix(tui): bind home/end to line start/end in input

- **Repo:** sst/opencode
- **PR:** https://github.com/sst/opencode/pull/25355
- **HEAD SHA:** `d3bda7e`
- **Author:** euxaristia
- **Verdict:** `merge-after-nits`

## What the diff does

Reshuffles the default keybindings for `home`/`end` (and `shift+home`/`shift+end`) inside the TUI input field so they behave like every other text editor on the planet — line-local navigation, not buffer-spanning navigation. The full set of changes is mirrored across ~25 localized doc files; the actual schema lives in `packages/opencode/src/config/keybinds.ts:43-92`:

| Action | Before | After |
|---|---|---|
| `messages_first` | `ctrl+g,home` | `ctrl+g` (drop `home`) |
| `messages_last` | `ctrl+alt+g,end` | `ctrl+alt+g` (drop `end`) |
| `input_line_home` | `ctrl+a` | `ctrl+a,home` |
| `input_line_end` | `ctrl+e` | `ctrl+e,end` |
| `input_select_line_home` | `ctrl+shift+a` | `ctrl+shift+a,shift+home` |
| `input_select_line_end` | `ctrl+shift+e` | `ctrl+shift+e,shift+end` |
| `input_buffer_home` | `home` | `ctrl+home` |
| `input_buffer_end` | `end` | `ctrl+end` |
| `input_select_buffer_home` | `shift+home` | `ctrl+shift+home` |
| `input_select_buffer_end` | `shift+end` | `ctrl+shift+end` |

Also a tip-text update at `feature-plugins/home/tips-view.tsx:75-82` removing the (now-stale) "Press Home to jump to the beginning of the conversation" hint.

## Why it's right

- **Conforms to platform convention.** macOS Cocoa, Windows readline, every web `<textarea>`, Bash readline, and Emacs all treat unmodified `Home`/`End` as line-local. Buffer-spanning behavior is virtually always behind a `Ctrl` modifier (Windows convention) or `Cmd+Up/Down` (macOS). The prior bindings violated muscle memory in a way users would frequently lose work to (e.g. `Home` followed by typing replaces the wrong text).
- **Load-bearing detail at the conflict resolution.** `home` was previously an alias for *both* `messages_first` (in the messages pane) and `input_buffer_home` (in the input pane). The reshuffle decisively removes the alias from `messages_first` and demotes the input-pane meaning to `ctrl+home` so the conflict no longer exists. This is the right primitive — a single keystroke shouldn't have two meanings depending on focus, because TUI focus is often visually ambiguous.
- **Backward compatibility for existing emacs-style users**: `ctrl+a`/`ctrl+e`/`ctrl+shift+a`/`ctrl+shift+e` are preserved as the *first* token in each multi-binding entry, so users with muscle memory for those don't regress. The new `home`/`end` are *additions* on the line-local actions, *removals* on the buffer-wide ones.
- **The doc/i18n update is consistent across all 25 locale files** (visible: `ar`, `bs`, `da`, `de`, `zh`, `zh-tw`, …) — no half-translated state.

## Nits

1. **The `messages_first`/`messages_last` removal of `home`/`end`** is technically a behavior change for users who relied on the messages-pane scroll-to-top via `Home`. Worth calling out explicitly in the PR description and the next changelog entry. Users who liked the prior binding can re-add it via `tui.json` — but they have to know to.
2. **No test for the binding parser.** The `keybind("ctrl+a,home", ...)` syntax is comma-separated multi-binding; if there's any place that expected exactly one binding per action, this PR would surface that bug at runtime. Adding a snapshot test of the resolved keybind map (or even just the schema validation passing) would lock the parser shape.
3. **Tips view at `tips-view.tsx:75-82`** still says "Press {highlight}Ctrl+G{/highlight} to jump to the beginning" — but doesn't mention `Ctrl+Home`/`Ctrl+End` for buffer-wide input navigation, which is the *new* binding users will discover. Add a tip for the new combination so users learn the new home for the old behavior.
4. **macOS users typically expect `Cmd+Left`/`Cmd+Right` for line-local and `Cmd+Up`/`Cmd+Down` for buffer-wide.** This PR (correctly) doesn't change those; but a follow-up adding the `cmd+`-modified bindings as additional aliases on macOS would make the input field feel native. Out of scope here, worth a tracking issue.
5. **25-file mechanical doc update** — would have been cleaner as a generated artifact (script that emits localized keybinds.mdx from the schema). Without that, the next keybind change has the same 25-file fanout. Tracking issue if not already filed.
6. **Spec compliance for `ctrl+shift+home`/`ctrl+shift+end`** — verify the underlying TTY input parser actually distinguishes `shift+ctrl+home` from `ctrl+home` on common terminals (xterm, iTerm2, Windows Terminal). Some terminal emulators flatten the modifier set and the binding may silently never fire.

`merge-after-nits` — right convention (line-local on `Home`/`End`, buffer-wide behind `Ctrl`), correct conflict resolution between messages-pane and input-pane, full i18n sweep, but wants a follow-up tip-text update and a terminal-compat note before users discover `ctrl+shift+end` doesn't fire on their TTY.
