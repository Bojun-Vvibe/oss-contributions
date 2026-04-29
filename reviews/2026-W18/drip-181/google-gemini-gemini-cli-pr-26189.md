# google-gemini/gemini-cli#26189 — fix(cli): prevent Windows bash backspace from triggering delete-word

- **Repo**: google-gemini/gemini-cli
- **PR**: [#26189](https://github.com/google-gemini/gemini-cli/pull/26189)
- **Head SHA**: `27f0cd191e4536ad8aa22f4b7e8d50183be0db13`
- **Author**: dreamaeiou
- **Diff stats**: +6 / −1 (1 file)

## What it does

On Windows-flavoured bash terminals (Git Bash, MSYS2), a plain
Backspace press is delivered to Node as the escape sequence
`ESC + \x7f` (`\x1b\x7f`). The existing `emitKeys` generator in
`KeypressContext.tsx` interpreted any character preceded by an `ESC`
in the buffer as `alt=true`. So users on Git Bash hitting Backspace
saw their keystroke routed to `DELETE_WORD_BACKWARD` (the binding
for `Alt+Backspace`) instead of `DELETE_CHAR_LEFT`. Result: pressing
Backspace ate entire words.

Fix: distinguish the genuine `Alt+Backspace` (which arrives as
`ESC + \b`, byte `\x08`) from the Git Bash plain-Backspace artifact
(`ESC + \x7f`). Only set `alt = true` when the `\b` form is seen with
an ESC prefix.

## Code observations

- `packages/cli/src/ui/contexts/KeypressContext.tsx:655-663` — original
  line `alt = escaped;` becomes
  `alt = escaped && ch === '\b';`. The condition reads as "alt is set
  only for the `\b` byte preceded by ESC, never for the `\x7f` byte".
  That matches the standard convention: `\b` (`^H`) is what `Alt+`
  modifier sequences emit; `\x7f` (`DEL`) is what bare Backspace emits
  on most modern terminals. Git Bash apparently emits `ESC + \x7f` for
  bare Backspace, which is the source of the false positive.
- The 4-line code comment is exemplary — it names the affected
  terminals (Git Bash, MSYS2), states what byte sequence they emit
  (`ESC + \x7f`), names the wrong action (`DELETE_WORD_BACKWARD`)
  vs. the right one (`DELETE_CHAR_LEFT`), and explains the
  asymmetric condition. Future-maintenance friendly.
- The fix is **asymmetric**: it accepts `ESC + \x7f` as plain
  backspace, but if any terminal sends `ESC + \b` for plain
  backspace (none of the major ones do, but it's worth a thought)
  the user would now get `Alt+Backspace` semantics. The PR's
  asymmetry choice is correct because the real-world distribution
  of "what byte a terminal sends for bare Backspace" is overwhelmingly
  `\x7f` and "what it sends for `Alt+Backspace`" is overwhelmingly
  `ESC + \b`. But it would be worth confirming that no observed
  terminal sends `ESC + \b` for plain Backspace before merging.

## Concerns

- **No regression test**. The byte-stream parser in `emitKeys` is
  testable — feed a `Buffer.from([0x1b, 0x7f])` and assert the
  yielded key has `name === 'backspace'` and `alt === false`. Without
  a test, the next refactor of this function (the area is dense and
  the conditions are easy to confuse) will silently regress this fix.
- **Doesn't address the dual case**: what about `Ctrl+Backspace`?
  On Windows terminals that's often `\x7f` alone or
  `ESC + \x7f` depending on the terminal. The fix as written says
  `ESC + \x7f` is plain Backspace, period — which means
  `Ctrl+Backspace` on Git Bash, if it's also `ESC + \x7f`, also
  falls into the plain-Backspace path. The user-facing impact is
  the same as before (one of the two Backspace variants didn't
  delete-word), so no new regression, but worth a comment that
  this fix is "plain Backspace correct; Ctrl+Backspace TBD".
- **Discoverability**: the `ch === '\b'` literal is a magic
  character. A named constant `BACKSPACE_CTRL_H = '\b'` (mirroring
  `ESC` already at `:660`) and `BACKSPACE_DEL = '\x7f'` would make
  the intent self-documenting and let a regression-test file import
  the constants instead of duplicating literals.

## Verdict: `merge-after-nits`

Correct, well-commented, addresses a real and frequently-reported
Git Bash UX bug. Nits:

1. Add a regression test in
   `packages/cli/src/ui/contexts/KeypressContext.test.ts` (or
   wherever `emitKeys` is tested) that feeds the four interesting
   sequences — `\x7f` alone, `\b` alone, `ESC + \x7f`, `ESC + \b` —
   and asserts the yielded `{ name, alt }` for each. Without this,
   a future refactor of `emitKeys` will silently re-break this.
2. Extract `'\b'` and `'\x7f'` into named constants
   (`BACKSPACE_CTRL_H`, `BACKSPACE_DEL`) at the top of the file,
   alongside the existing `ESC` constant, so the condition reads
   `alt = escaped && ch === BACKSPACE_CTRL_H`.
3. Note in the comment block whether `Ctrl+Backspace` on Git Bash
   is in scope or out of scope. If it produces the same `ESC + \x7f`
   bytes, this fix necessarily folds Ctrl+Backspace into plain
   Backspace, and that should be explicit.
4. Verify on Windows Terminal + WSL bash, Git Bash, MSYS2 bash, and
   PowerShell that the four interesting Backspace combinations
   produce the expected behavior. List the test matrix in the PR
   body.

## Follow-ups

- File a meta-issue tracking "TUI key-event correctness on
  Windows-bash terminals" with a checklist of every modifier+key
  combination the readline / TUI code handles. Backspace is the
  most-pressed; arrow keys, `Ctrl+Left`/`Ctrl+Right`, and
  `Alt+.` (last-arg recall) are the next most likely to have
  similar artifacts.
