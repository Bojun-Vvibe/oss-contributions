# Review: google-gemini/gemini-cli#26270 — feat(ui): added microphone and updated placeholder for voice mode

- PR: https://github.com/google-gemini/gemini-cli/pull/26270
- Head SHA: `f8237d1feb78e6709a5839cfff5e6c51dc0bd6b6`
- Files: ~3 (+44/-38)
- Merged: 2026-04-30
- Closes: #25490

## Stated purpose
When the user enables voice mode (`/voice`), the existing UX surfaced a `Voice mode: Space to start/stop recording` hint and a `🎙️ Listening...` indicator only while recording. This left the input prompt looking *identical* to text mode when the user wasn't actively recording — users would type, hit enter, and get confused by their typed text being submitted instead of converted-to-voice. This PR makes the voice-mode-active state visually persistent via a microphone glyph (`🎤 >`) prefix on the prompt, and rewords the hint to communicate that both typing and talking are valid actions.

## What actually changed
The diff visible in `InputPrompt.test.tsx` (the only file shown in the partial diff) shows the contract changes that the underlying component must now satisfy:
1. Idle voice-mode state changes from showing `Voice mode: Space to start/stop recording` (a discoverable but easy-to-miss footer hint) to showing `🎤 >` as a prompt prefix plus `Type your message or space to talk (Esc to exit)` placeholder text.
2. Recording-state indicator changes from `🎙️ Listening...` (studio-mic glyph) to `Listening...` with the mic-glyph presumably moved to the prompt prefix during recording too — the test only asserts substring presence so the exact rendering can vary.
3. The "voice mode hint persists when buffer is non-empty" path is preserved (line ~5046) — important UX detail: a partially-typed message shouldn't suddenly hide the voice-mode affordance.
4. The `handleInput`-fallthrough test at line ~5082 confirms that space-as-input (e.g. typing a space inside an existing message) still routes to `handleInput` rather than triggering recording, when the buffer has content.

## Quality / risk observation
The UX direction is right — moving the mode indicator from a footer hint to a *prompt prefix* makes the mode state part of the user's eye-line at every keystroke, which is the correct visual weight for a modal state that changes input semantics. The placeholder rewording from "Space to start/stop recording" to "Type your message or space to talk (Esc to exit)" is also better — it surfaces all three actions (type, talk, exit) in one line and uses `Esc to exit` which matches the standard escape-modal convention used elsewhere in the UI. One observation about test coverage: the test diff only shows substring assertions on `lastFrame()` — it doesn't pin the *position* of the `🎤 >` prefix relative to the buffer content. If a future contributor changes the prompt to render the glyph as a *suffix* (e.g. `> 🎤 my message`) the test would still pass but the UX would regress. A `toMatch(/^.*🎤 >.*$/m)` or similar position-aware assertion would tighten the regression net. Second observation: switching from `🎙️` (studio microphone) to `🎤` (handheld microphone) is a deliberate aesthetic choice but worth confirming with the design team — both render in most monospace terminals but the glyphs have meaningfully different cell widths in some fonts, and the handheld-mic is sometimes rendered as a colour emoji with VS16 selectors that double the apparent width and break alignment in the prompt prefix. Third: the test `not.toContain('Listening...')` substring after stop-recording is a slightly weak assertion — `Listening...` is a common string that could legitimately appear in error messages or other UI text; `not.toContain('🎤 Listening...')` (with the glyph) would be more specific. Pure UX polish PR; zero functional risk; ship after the position-pinning test nit.

## Verdict
`merge-after-nits`
