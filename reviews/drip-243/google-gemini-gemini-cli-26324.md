# Review: google-gemini/gemini-cli #26324 — fix(cli): prevent ghost text wrapping infinite loop

- **PR**: https://github.com/google-gemini/gemini-cli/pull/26324
- **Author**: vyctorbrzezowski (Vyctor Huggo Przozwski)
- **Head SHA**: `f68913c4d5b978b7347a37c033f5d04c62387d69`
- **Base**: `main`
- **Files**: `cli/src/ui/components/InputPrompt.tsx` (+40/-20),
  `cli/src/ui/components/InputPrompt.test.tsx` (+17/-0)
- **Verdict**: **merge-as-is**

## Reasoning

Fixes #19985 — the interactive CLI hangs ("ghost text wrapping infinite
loop") when the prompt input contains a long unbroken word like
`@getskill.sh:3` and the available terminal width is zero or narrower
than the next code-point's display width.

Root cause analysis encoded in the diff:

The pre-fix logic at `InputPrompt.tsx:~1485-1505` had this shape:

```ts
while (stringWidth(wordToProcess) > inputWidth) {
  // accumulate codepoints until partWidth + charWidth > inputWidth, then break
  // push 'part', then wordToProcess = cpSlice(wordToProcess, splitIndex);
}
```

The hang case is `inputWidth === 0` (or a wide character with `charWidth
> inputWidth`): the inner `for` loop hits `partWidth + charWidth > width`
on the very first iteration, breaks immediately with `splitIndex === 0`,
and the outer `while` loop pushes an empty `part`, slices zero codepoints
off, and re-tests the same `wordToProcess` against the same `inputWidth`
forever.

The fix at `InputPrompt.tsx:162-198` extracts the wrapping logic into an
exported `splitWordForWidth(word, maxWidth)` helper with two correctness
guards:

1. `const width = Number.isFinite(maxWidth) && maxWidth > 0 ? maxWidth : 1`
   — clamps the working width to at least 1, neutralizing the
   `inputWidth === 0` case at the entry point.
2. The inner loop now has the load-bearing
   `if (partWidth === 0) { part += char; splitIndex = i + 1; }` clause
   before `break`, which guarantees forward progress when a *single*
   character is wider than the available width: instead of breaking
   with a zero-codepoint slice, it consumes that one character and
   then breaks. So `splitIndex >= 1` always, and the outer `while` is
   guaranteed to terminate.

The call site at `InputPrompt.tsx:1521-1525` is rewritten to consume the
helper:

```ts
const splitWord = splitWordForWidth(word, inputWidth);
additionalLines.push(...splitWord.slice(0, -1));
currentLine = splitWord.at(-1) ?? '';
```

This preserves the contract that the *last* fragment becomes the new
`currentLine` (continuing to be filled with subsequent words), and all
prior fragments are flushed as full lines. The `?? ''` fallback handles
the edge case where `splitWord` is somehow empty.

Tests at `InputPrompt.test.tsx:484-497` cover the two killer cases:

- `splitWordForWidth('@getskill.sh:3', 0)` — width zero, asserts every
  codepoint is preserved (`lines.join('') === '@getskill.sh:3'`) and
  the split is per-codepoint (`'@getskill.sh:3'.split('')`).
- `splitWordForWidth('表x', 1)` — wide CJK character into a width-1
  budget, asserts the wide character occupies its own line and the
  ASCII char follows.

Both tests would have hung indefinitely against the pre-fix code, so
they double as regression fences.

## Suggested follow-ups

- Optional: add a fuzz test or property-based test asserting the
  invariant `splitWordForWidth(any, any).join('') === any` for arbitrary
  unicode inputs and width values. The two unit tests cover the
  reported failure mode but not the broader correctness property.
- The `width = ... > 0 ? maxWidth : 1` clamp should probably get a
  one-line comment explaining *why* the floor is 1 rather than e.g. 0
  or `inputWidth`. Pure documentation, not blocking.
- Consider exporting `splitWordForWidth` from a shared text-layout
  utility module if other input paths grow similar wrapping needs;
  having it in the `InputPrompt.tsx` namespace is fine for now.
