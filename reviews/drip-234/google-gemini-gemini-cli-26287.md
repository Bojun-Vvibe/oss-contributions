# google-gemini/gemini-cli#26287 — fix(cli): insert voice transcription at cursor position instead of appending

- **PR**: https://github.com/google-gemini/gemini-cli/pull/26287
- **Head SHA**: `d961089d0d838aa98faae088eacb3c01aa222000`
- **Size**: +62 / -31, 2 files
- **Verdict**: **merge-after-nits**

## Context

Voice transcription in the CLI input prompt previously appended the live transcription to the end of the buffer (`bufferRef.current.setText(newTotalText, 'end')`). If the user had already typed `hello world` and placed their caret after `hello`, then enabled voice and said "there", the result was `hello world there` (appended) rather than `hello there world` (inserted at caret). This PR fixes that to insert at the caret position captured at the moment voice activation began.

## Design analysis

### State capture (`useVoiceMode.ts:54, 113-115`)

New ref:
```ts
const turnBaselineCursorOffsetRef = useRef<number>(0);
```

Captured alongside the existing `turnBaselineRef` at the start of recording:
```ts
turnBaselineRef.current = bufferRef.current.text;
turnBaselineCursorOffsetRef.current = bufferRef.current.getOffset();
```

This is the load-bearing decision: snapshot **the caret position at recording-start**, not the live caret. Live caret would move as text is inserted, which would create a runaway "transcription pushes cursor, next delta inserted at new position" loop. Snapshotting freezes the insertion point for the entire turn.

### Insertion logic (`useVoiceMode.ts:197-216`)

```ts
const baseline = turnBaselineRef.current ?? '';
const insertOffset = turnBaselineCursorOffsetRef.current;
const textBefore = baseline.slice(0, insertOffset);
const textAfter = baseline.slice(insertOffset);

const prefix =
  textBefore.length > 0 && !/\s$/.test(textBefore)
    ? textBefore + ' '
    : textBefore;

const suffix =
  text.length > 0 && textAfter.length > 0 && !/^\s/.test(textAfter)
    ? ' '
    : '';

const newTotalText = prefix + text + suffix + textAfter;
bufferRef.current.setText(newTotalText, prefix.length + text.length);
```

Three correctness pieces:
1. **`baseline` (frozen) not `bufferRef.current.text` (live)** — this means re-running the transcription handler with an updated `text` re-derives the buffer from the original baseline rather than re-appending. Old code used the live `currentBufferText` and tried to detect "is the previous transcription still at the end" via `endsWith` — fragile and broke on cursor-not-at-end.
2. **Whitespace inference**: prefix gets a trailing space only if `textBefore` is non-empty and doesn't already end in whitespace. Suffix gets a leading space only if `text` is non-empty AND `textAfter` is non-empty AND `textAfter` doesn't start with whitespace. This handles "hello|world" → "hello there world" correctly (one space each side because neither side has natural whitespace at the join point).
3. **Caret position**: `setText(newTotalText, prefix.length + text.length)` — the second arg is the new caret offset, exactly at the end of the inserted transcription, *before* the suffix and after-text. Caret stays "in" the just-spoken text.

### Turn baseline advance (`useVoiceMode.ts:223-226`)

On `turnComplete`:
```ts
turnBaselineRef.current = bufferRef.current.text;
turnBaselineCursorOffsetRef.current = bufferRef.current.getOffset();
```

This is the second load-bearing decision: when Gemini Live signals end-of-turn, baseline advances to the new buffer + the **current** cursor (which is now after the just-inserted transcription). The next `transcription` event for a new turn appends after the prior turn rather than overwriting.

### Test additions (`InputPrompt.test.tsx`)

- The mock `getOffset` at `:351` is changed from a fixed `0` return to `vi.fn().mockImplementation(() => mockBuffer.cursor[1])` — now the mock buffer's caret position is real.
- Existing test arm at `:5114-5135` updated: `'end'` → numeric offset `13` (length of "initial hello") and `19` (length of "initial hello world"). Asserts the actual numeric caret position rather than the symbolic `'end'`. This makes the test pin the caret-positioning contract precisely.
- Existing test at `:5169-5174` updated similarly to assert offset `24` ("First turn. Second turn." length).
- **New test arm at `:5180-5210`** (`should insert transcription at cursor position when buffer has text before and after (toggle)`): pre-sets `mockBuffer.setText('hello world')` and `mockBuffer.cursor = [0, 5]`, fires transcription `'there'`, asserts result `('hello there world', 11)`. The 11 is computed in the comment: "'hello'(5) + ' '(1) + 'there'(5) = cursor at 11; ' world' preserved after". Pins both the inserted text and the caret position.

## What's right

- **Frozen-baseline pattern.** Eliminates the entire class of "live-buffer-state diff" bugs that the old `currentBufferText.endsWith(previousTranscription)` heuristic was vulnerable to. Live Gemini revises its transcription mid-turn (e.g. "hello" → "hello world" → "hello world how are you"); the new code re-derives from baseline + latest text every time, so revisions cleanly replace, not accumulate.
- **Whitespace inference is symmetric.** Prefix-space and suffix-space both check the join character, not just one side.
- **`turnComplete` advances baseline.** Multi-turn transcription (e.g. user pauses, resumes) appends rather than overwrites. The test at `:5169-5174` pins this contract.
- **Caret lands inside the just-spoken text.** Important UX — user can immediately backspace-correct the transcription without re-positioning.
- **Mock buffer's `getOffset` now reflects `cursor[1]`** so the tests exercise the actual computation rather than a fixed 0.

## Nits

- **`getOffset()` returns column on the cursor's line.** The mock returns `mockBuffer.cursor[1]` (column index). For multi-line buffers, the real `getOffset()` likely returns a flat character offset across all lines. The new test only covers single-line buffers — worth a multi-line test arm to confirm the slicing math (`baseline.slice(0, insertOffset)`) handles `\n` correctly. Specifically: if the user has a multi-line draft and places the caret on line 3, does `baseline.slice(0, getOffset())` correctly produce the prefix including all earlier lines and their `\n`? Probably yes (`getOffset` on a real `TextBuffer` is the flat offset), but without a test the answer is "looks right by inspection".
- **`text.length > 0` check in suffix.** If `text` is empty (initial connecting state, no speech yet), suffix is empty — but the entire inserted-text branch is gated on `if (text)` at `:197`, so this inner check is defensive-only. Fine.
- **Tab character in `textBefore`.** `/\s$/` matches `\t`, `\n`, ` ` — anything Unicode-whitespace. Symmetric with `/^\s/` on `textAfter`. Correct.
- **No test for the "user moves cursor *during* recording"** case. Spec is "snapshot at start"; user moving the cursor mid-recording shouldn't change where the transcription lands. The frozen-baseline pattern guarantees this by construction, but a test arm would pin it.
- **`liveTranscriptionRef.current = text;`** at `:217` is preserved from the old code but its only consumer (the old `previousTranscription` check) was removed. Now the assignment is dead code — could be removed. Minor.

## Verdict

**merge-after-nits** — surgical fix to a real UX bug, with the right architectural pattern (frozen baseline + insert-at-snapshotted-offset) replacing the fragile live-buffer-diff heuristic. Test coverage is solid for single-line cases. Multi-line + mid-recording-cursor-move test arms would close the remaining edge cases. Dead `liveTranscriptionRef.current` assignment can be cleaned up.
