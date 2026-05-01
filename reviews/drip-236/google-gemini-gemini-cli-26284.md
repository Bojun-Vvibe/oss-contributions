# google-gemini/gemini-cli#26284 — feat(ui): added wave animation for voice mode

- **PR**: https://github.com/google-gemini/gemini-cli/pull/26284
- **Head SHA**: `5f77d6ad141e06ddcf3dfd46c2844bac2844cfb3`
- **Size**: +54 / -7, 3 files
- **Verdict**: **merge-after-nits**

## Context

Closes #25493. Replaces the static `🎤` glyph + `Listening...` plain-text indicator inside `InputPrompt` with a 3-bar sine-wave animation (`ListeningIndicator.tsx`) when the user is actively recording in voice mode. The static `🎤` is preserved for the voice-mode-enabled-but-not-recording state.

## What's right

**`packages/cli/src/ui/components/shared/ListeningIndicator.tsx` (new, 37 lines)** — minimal, self-contained component:

- Single `useState(0)` tick counter incremented every 80ms via `setInterval` (12.5 FPS, smooth enough for terminal rendering without flickering, slow enough to not burn CPU).
- `useEffect` cleanup at `:21-25` properly clears the interval on unmount via `return () => clearInterval(timer)`.
- 3 bars generated via `Array.from({ length: 3 }).map((_, i) => ...)`. Phase computed as `tick * 0.4 + i * 1.2` per bar, mapped through `Math.floor((Math.sin(phase) + 1) * 3.5)` to a 0–7 index into `WAVE_CHARS = [' ', '▂', '▃', '▄', '▅', '▆', '▇', '█']`.
- Defensive `Math.max(0, Math.min(7, height))` clamp at `:31` plus `?? ' '` fallback at `:32` so out-of-range or undefined indexes degrade to a space rather than throwing.

The `0.4` and `1.2` constants control the wave's temporal frequency and per-bar phase offset respectively; they produce a visually pleasant out-of-phase wave where each bar leads/lags its neighbor by ~115° (1.2 rad). Reasonable defaults; would benefit from a one-line comment explaining the geometric intent.

**`InputPrompt.tsx:1801-1811` — recording-state branch.**

```diff
- {isVoiceModeEnabled && <Text color={theme.text.accent}>🎤 </Text>}
+ {isVoiceModeEnabled &&
+   (isRecording ? (
+     <ListeningIndicator color={theme.text.accent} />
+   ) : (
+     <Text color={theme.text.accent}>🎤 </Text>
+   ))}
```

Correctly preserves the prior `🎤` glyph for the voice-mode-on-but-not-recording state, swapping in the wave animation only when actively recording. The `isRecording` predicate is the same one driving the `Listening...` text below.

**`InputPrompt.test.tsx:108-110` — `vi.mock('./shared/ListeningIndicator.js', ...)`** stubs the animation to a deterministic `~~~ ` string so existing snapshot/text tests don't fight the live `setInterval`. The matching update to the test assertion at `:5291` (`'🎤 >'` → `'~~~ >'`) keeps the existing test passing under the new component.

## Risks / nits

- **`Listening...` text is duplicated.** Diff at `InputPrompt.tsx:1815-1820` adds `{isRecording && (<Text color={theme.status.success}>Listening... </Text>)}` *outside* the inner `Box flexDirection="column"`, while *also* removing the prior `<Box flexDirection="row" marginBottom={0}><Text color={theme.status.success}>Listening...</Text></Box>` block from inside the inner box. Net: the `Listening...` text moves from inside the column to outside it, becoming inline with the wave indicator. This is the intended layout change, but the diff would be much clearer as a single move-the-block edit rather than the current add-then-delete shape.
- **Bug at `InputPrompt.tsx:1828-1833` — duplicate `showYoloStyling ? '*'` arm in the ternary chain.** The diff hunk reads:
  ```jsx
  ) : showYoloStyling ? (
    '*'
  ) : showYoloStyling ? (
    '*'
  ) : (
    '>'
  )}{' '}
  ```
  The second `showYoloStyling ? '*'` arm is unreachable and almost certainly a merge or rebase artifact. It compiles and behaves correctly (the second arm is dead code), but a future linter pass with `no-duplicate-conditions` enabled would flag it. **Should be removed before merge.**
- **`tick` counter is `useState<number>` and increments without bound** every 80ms. At 12.5 increments/sec, hitting `Number.MAX_SAFE_INTEGER` (2^53) takes ~22 million years, so practically not a concern, but a recording session left open for days will eventually start losing precision in `tick * 0.4` (around 2^45 ≈ 1100 years in). Not a real issue; mention only because the obvious "fix" of `setTick(t => (t + 1) % SOME_PERIOD)` would be wrong (would cause visible jumps when wrapping). Leave as-is.
- **`setInterval(80ms)` re-renders the component every frame regardless of focus.** Solid for short recordings; if voice mode permits long-form dictation (>1 minute), a `useEffect`-based opt-out for backgrounded terminals would save CPU. Not blocking — the predicate `isRecording` already gates whether `<ListeningIndicator>` is even mounted, so the timer only runs during active recording.
- **No accessibility consideration.** The component uses no `aria-label`. Screen-reader users in voice mode get no indication of the recording state from the bar animation alone; the adjacent `Listening...` text remains as the screen-reader-friendly cue, but if a downstream change ever drops it, the visual-only indicator becomes inaccessible. Worth a one-line `<Text aria-label="Listening">{bars.join('')} </Text>` (Ink supports this) or a comment that screen-reader users rely on the sibling text.
- **`80ms` interval and the `0.4` / `1.2` magic numbers** would benefit from a 2-line comment naming them as "frequency = 0.4 rad/tick" and "phase offset = 1.2 rad/bar" so a future contributor tweaking the animation knows which knob does what.
- **`WAVE_CHARS` includes both `' '` (space) at index 0 and the 7 block-fill glyphs.** When `Math.sin(phase) + 1 = 0` and `* 3.5 = 0` and `Math.floor` returns `0`, the bar renders as a space — meaning the 3-bar visual occasionally has a fully-empty bar in the middle. That's the intended sine-wave bottom-of-cycle behavior, but worth confirming visually in PR (the linked screen capture in the PR body does show this).

## Verdict

**merge-after-nits.** Clean self-contained UI component with sensible defaults and proper cleanup. The `isRecording`-gated mount keeps the timer scope tight. **Must fix the duplicate `showYoloStyling ? '*'` arm before merge** — that's a real diff bug, not a stylistic concern. The other items (geometric-constant comments, accessibility note, screen-reader fallback) are nice-to-have polish.
