# sst/opencode #25830 — docs: Document Shift+Enter key binding for Alacritty and Tmux

- **Head SHA:** `30e31bbd52f5f09396a2148963ad255ff3875898`
- **Author:** @angelcervera
- **Verdict:** `merge-after-nits`
- **Files:** +21 / −0 in `packages/web/src/content/docs/keybinds.mdx`

## Rationale

Pure docs add to `packages/web/src/content/docs/keybinds.mdx:170-188` extending the existing "Some terminals don't send modifier keys with Enter by default" section with concrete recipes for Alacritty (`alacritty.toml` keyboard binding using the CSI-u sequence `\u001b[13;2u`) and Tmux (`xterm-keys`/`extended-keys`/`allow-passthrough` settings). These are exactly the right two terminals to call out — Alacritty is a top-3 destination for power users and Tmux is the universal multiplexer that breaks every "Shift+Enter just works" expectation. The sequence `\u001b[13;2u` is the correct CSI-u encoding for Shift+Return (modifier mask 2 = Shift, codepoint 13 = Return), and the Tmux trio is the canonical pass-through recipe.

**Nits, in order of importance:**

1. **Typo: "key bindig" appears twice** — once on line 173 (Alacritty) and once on line 184 (Tmux) of the post-PR file. Should be "key binding". This is the single most important fix; it'll be the first thing readers notice.
2. **`allow-passthrough on` may be misleading guidance.** Tmux's `allow-passthrough` controls DCS pass-through (escape sequences from inside-pane programs reaching the outer terminal), not modified-key reporting. The two settings that actually matter for Shift+Enter are `xterm-keys on` (Tmux 1.x compatibility flag — actually a no-op on modern Tmux 3.x where it defaults on) and `extended-keys on` (the real switch that enables CSI-u modified-key encoding). I'd drop `allow-passthrough` from this recipe or at least add a sentence explaining what it does — leaving it in suggests it's required when it isn't, and users who copy-paste configs without understanding them end up with a permanently-on pass-through they didn't ask for.
3. **Tmux version requirement.** `extended-keys` landed in Tmux 3.2 (2021). Worth a half-sentence: "(Tmux 3.2+)".
4. **Consider linking to the upstream CSI-u spec** (or to `xterm`'s `modifyOtherKeys` doc) so power users who want to know *why* `\u001b[13;2u` is the correct escape have somewhere to go.

Net: clear improvement, fix the typos and ideally trim/explain `allow-passthrough` before landing. No risk to ship — it's an MDX file with no code consequences.
