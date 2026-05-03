# google-gemini/gemini-cli #26278 — feat(voice): add dynamic audio wave animation for recording feedback

- **PR**: https://github.com/google-gemini/gemini-cli/pull/26278
- **Head SHA**: `91d35ba75a66fff1922ccdaceef0ab2f65ae3f8d`
- **Author**: fauzan171
- **Size**: +92 / −2, 2 files

## Files changed

- `packages/cli/src/ui/components/AudioWaveIndicator.tsx` (+89, new)
- `packages/cli/src/ui/components/InputPrompt.tsx` (+3 / −2)

## Summary of change

Adds a new Ink component that renders an animated 8-bar audio wave using
Unicode block characters (`▁▂▃▄▅▆▇█`) to indicate that voice recording
is active. Falls back to the literal text `"Recording audio..."` when a
screen reader is enabled. Wires it into `InputPrompt.tsx` (3-line edit).

## Specific-line review

- `AudioWaveIndicator.tsx:14-18` — constants `BAR_COUNT = 8`,
  `FRAME_INTERVAL_MS = 66` (~15 fps), `CYCLE_DURATION_MS = 1200`. The
  15 fps cap is sensible for a TTY that re-renders the whole frame on
  every `setState`; bumping any higher would risk thrashing low-end
  terminals.
- `AudioWaveIndicator.tsx:39-49` — `useEffect` short-circuits when
  `useIsScreenReaderEnabled()` is true (no interval scheduled). Good:
  this avoids burning CPU on a feature the user can't see, and avoids
  spamming a screen reader with re-renders. The cleanup
  `return () => clearInterval(interval)` is present — no leak on unmount.
- `AudioWaveIndicator.tsx:51-53` — screen-reader fallback returns a
  static `<Text>Recording audio...</Text>`. This is the right ARIA
  affordance for a TTY UI; the only nit is that "Recording audio..."
  will be announced once and then go silent forever, so there is no
  auditory cue that recording is *still* active. Consider an aria-live
  equivalent or periodic re-announcement — but Ink doesn't really
  expose live-region semantics, so this is probably fine as-is.
- `AudioWaveIndicator.tsx:60-65` — connecting state: `dotCount` cycles
  1→2→3 every 4 frames (~264 ms per step). Reasonable pulse rate.
- `AudioWaveIndicator.tsx:69+` (cut off in diff sample) — sinusoidal
  bar generation with phase offset per bar. I'd want to spot-check the
  bar-index → phase math to make sure there is no unbounded `frame`
  accumulation issue. `frame` is plain `useState(number)` and
  monotonically increments at 15 fps; over a long-running session with
  voice always on this will eventually drift into very large integers,
  but JS `number` handles that fine and `(frame * FRAME_INTERVAL_MS) %
  CYCLE_DURATION_MS` keeps the visible state bounded. No real risk.
- `InputPrompt.tsx` (+3 / −2) — small wiring change to render the
  indicator when the input mode is recording. The diff is too short to
  introduce regressions in non-voice flows.

## Things to consider (non-blocking nits)

- `🎙️` emoji in the connecting state may render as `??` or a tofu box
  on terminals without emoji support. The wave glyphs (`▁`–`█`) are CP437
  / box-drawing-adjacent and have much wider terminal coverage. Consider
  guarding the emoji behind the same `useIsScreenReaderEnabled` check
  or a "supports unicode emoji" capability — or just drop it.
- A Vitest snapshot or `ink-testing-library` test that asserts the
  fallback text appears under screen-reader mode and the wave string is
  non-empty otherwise would lock in both code paths cheaply.

## Verdict

**merge-after-nits** — feature is well-scoped, accessibility branch is
present, no obvious leaks. Worth asking the author for one render test
and a brief consideration of the emoji-fallback question before merge.
