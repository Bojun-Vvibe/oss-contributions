# sst/opencode PR #25355 — fix(tui): bind home/end to line start/end in input

- URL: https://github.com/sst/opencode/pull/25355
- Head SHA: `d3bda7e85b1b41c05e6d6e9f9754e7ccf4b76f17`
- Author: euxaristia
- Verdict: **merge-after-nits**

## Summary

Reworks the default keybinding scheme so that the bare `Home` / `End` keys (and their `Shift+` variants) operate on the current *line* of the input rather than on the whole *buffer*, matching macOS/Windows/Linux text-editing convention. Buffer-level navigation moves to `Ctrl+Home` / `Ctrl+End`. The conversation-level `messages_first`/`messages_last` shortcuts are also stripped of their `home`/`end` aliases (now `ctrl+g` / `ctrl+alt+g` only) to free those keys for the input. Tip strings and all localized keybind docs are updated in lockstep.

## Line-level observations

- `packages/opencode/src/config/keybinds.ts` lines 46–47: `messages_first` and `messages_last` lose their `,home`/`,end` second binding. Anyone who muscle-memorized `Home` to jump to the conversation start (e.g. from earlier docs) will silently get a different behavior — now jumps to the line start of the input. Worth calling out in the PR description / changelog.
- `packages/opencode/src/config/keybinds.ts` lines 79–82 and 88–91: the new `,home`/`,end` aliases are appended after the existing `ctrl+a`/`ctrl+e` etc., which is the correct order — existing emacs-style bindings continue to work.
- `packages/opencode/src/cli/cmd/tui/feature-plugins/home/tips-view.tsx` lines 78–79: the help tip is correctly truncated to drop the `or Home` / `or End` mention. Good — would otherwise contradict the new defaults.
- The diff updates ~30+ localized `keybinds.mdx` files mechanically. I spot-checked `ar`, `bs`, `da` — all consistent. Reviewer should `grep -L "ctrl+home" packages/web/src/content/docs/*/keybinds.mdx` to confirm none were missed.
- No code changes to the actual key dispatcher were needed because the strings are already comma-split downstream; this is a pure config/data migration. Low risk.

## Suggestions

1. Add a one-line entry to the changelog / release notes: "Breaking: `Home` / `End` now navigate within the input line, not the buffer. Use `Ctrl+Home` / `Ctrl+End` for buffer ends; conversation-jump bindings are now `Ctrl+G` / `Ctrl+Alt+G` only."
2. Worth scripting the localized `.mdx` updates (or generating them from `keybinds.ts`) so the next change doesn't have to touch ~30 files by hand. Out of scope for this PR but a worthwhile follow-up.
3. Consider whether `Cmd+Home` / `Cmd+End` on macOS terminals (which often emit different escape sequences) need a separate alias — the current bindings only cover `ctrl+...`.
