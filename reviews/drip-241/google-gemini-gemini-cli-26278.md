# google-gemini/gemini-cli#26278 — feat(voice): add dynamic audio wave animation for recording feedback

- **PR**: https://github.com/google-gemini/gemini-cli/pull/26278
- **Head SHA**: `91d35ba75a66fff1922ccdaceef0ab2f65ae3f8d`
- **Files**: `packages/cli/src/ui/components/AudioWaveIndicator.tsx` (+89/-0, new), `packages/cli/src/ui/components/InputPrompt.tsx` (+3/-2)
- **Verdict**: **merge-after-nits**

## Context

Closes #25493 ("epic: voice mode UX polish"). Replaces the static `🎙️ Listening...` text in `InputPrompt.tsx` with a new `AudioWaveIndicator` React component that renders an animated 8-bar sinusoidal wave using Unicode block characters (`▁▂▃▄▅▆▇█`) at ~15fps, plus a pulsing `🎙️ Connecting...` state with animated dots while the transcription service is being established. Falls back to a simple "Recording audio..." text label when a screen reader is detected.

## What's right

- **Screen-reader fallback is the first thing checked.** `AudioWaveIndicator.tsx:51-54` short-circuits the `useEffect` interval setup if `isScreenReaderEnabled`, then `:65-67` returns a plain `<Text>Recording audio...</Text>` — animated bars would be noise to a screen reader. Correct accessibility discipline; the prior static `🎙️ Listening...` had nothing equivalent.
- **`isConnecting` state distinguishes "service is being established" from "actively recording".** `:69-77` shows the pulsing-dots variant during connection (1→2→3 dot cycle every 12 frames), `:80+` shows the wave during actual recording. Two clearly distinct visual states for two semantically distinct phases. The `isConnecting` field is destructured from `useVoiceMode` at `InputPrompt.tsx:288` — pre-PR it was unused; this PR is the first consumer.
- **Animation cleanup is correct.** `:53-58` `setInterval` with `return () => clearInterval(interval)` from the `useEffect` — no leaked timers when the component unmounts (e.g., user releases Space and recording stops).
- **Phase-offset wave generation is mathematically sound.** `:81-90`: `phase = (elapsed / CYCLE_DURATION_MS) * Math.PI * 2`, then per bar `barPhase = phase + (i / BAR_COUNT) * Math.PI`, mapped via `(Math.sin(barPhase) + 1) / 2` to `[0, 1]` and indexed into `BAR_LEVELS`. The half-period offset between bars (`Math.PI / BAR_COUNT * i`) produces the visual "wave traveling across the bars" effect. `Math.floor(normalized * (BAR_LEVELS.length - 1))` correctly maps to indices `[0, 7]` inclusive (no off-by-one — `normalized * 7` then floor gives `[0, 6]` for `[0, 1)` and `7` for exactly `1.0`).
- **Constants are named and documented.** `BAR_COUNT = 8`, `BAR_LEVELS = ['▁', ..., '█']`, `FRAME_INTERVAL_MS = 66` (with comment `~15fps for smooth feel`), `CYCLE_DURATION_MS = 1200` — all pinned at module scope with clarifying comments. No magic numbers in the rendering code.
- **`InputPrompt.tsx:288` and `:1813-1816`** changes are minimal: one import, one `useVoiceMode` destructure addition (`isConnecting`), one JSX swap. Easy to revert if needed.
- **Themed color via `Colors.AccentGreen` at `:71`/`:91`** — consistent with the prior static text using `theme.status.success` (now removed at `:1816`). Visual continuity preserved.

## Risks / nits

- **`useEffect` re-runs when `isScreenReaderEnabled` changes**, but the `setInterval` runs unconditionally at 15fps for the entire recording duration. On a long recording session this is ~900 React re-renders per minute. While Ink is generally efficient, this could noticeably load the terminal on lower-end systems. Recommend either: (a) reducing to ~10fps (`FRAME_INTERVAL_MS = 100`) which is still smooth enough for an indicator and reduces the React rerender cost by 33%, or (b) verifying empirically that 15fps is fine on a Raspberry Pi-class device. PR body's testing section doesn't mention perf.
- **`(frame % 12) < 4 ? 1 : (frame % 12) < 8 ? 2 : 3` at `:71`** is a clever inline ternary but slightly opaque; a small `getDotCount(frame)` helper or a comment "// 1 dot for frames 0-3, 2 for 4-7, 3 for 8-11, then cycles" would help future maintainers.
- **No unit tests added.** A snapshot test asserting the bar string at frame 0, frame 9 (~`CYCLE_DURATION_MS / 2 / FRAME_INTERVAL_MS`), and the connecting state at frame 0/4/8 would pin the intended visual contract. If something regresses in the math, the symptom would be "indicator looks weird" rather than a test failure.
- **`isConnecting` defaults to `false` at `:31`** but if `useVoiceMode` doesn't provide it (e.g., during partial rollouts), the indicator will skip the connecting state entirely and immediately show the wave. Fine for the new code path, but worth a defensive check — or, since the PR adds the destructure at `:288`, confirm `useVoiceMode` actually returns `isConnecting` (not `undefined`).
- **Emoji-plus-bold-text ordering at `:91-93`**: `🎙️ <Text bold>{bars}</Text>` may render the emoji and bars on visually different baselines on some terminals (terminal emoji width is famously inconsistent). Worth manual testing on at least Terminal.app, iTerm2, Windows Terminal, and Linux GNOME Terminal.
- **The `setFrame((prev) => prev + 1)` at `:55` will eventually overflow** to `Number.MAX_SAFE_INTEGER` after ~`9e15 / 15` ≈ 19 billion seconds (~600 years), so practically harmless, but `setFrame((prev) => (prev + 1) % 1_000_000)` would be defensively cleaner since the frame value is only used modulo small constants anyway.

## Verdict

**merge-after-nits.** Real UX win — animated wave is more informative than static "Listening..." text at zero accessibility cost (screen-reader fallback first), with separate `isConnecting` state for the service-establishment phase. Math is sound, animation cleanup is correct, constants are named. Strongest nits: add at least snapshot tests for the bar generation at known frames, consider 10fps instead of 15fps for terminal load, replace the inline `(frame % 12) < 4 ? ... : ...` ternary with a named helper or clarifying comment, and verify emoji+bar rendering across the major terminals.
