# google-gemini/gemini-cli #26189 — prevent Windows bash backspace from triggering delete-word

- **PR:** https://github.com/google-gemini/gemini-cli/pull/26189
- **Title:** `fix(cli): prevent Windows bash backspace from triggering delete-word`
- **Author:** dreamaeiou
- **Head SHA:** 27f0cd191e4536ad8aa22f4b7e8d50183be0db13
- **Files changed:** 1 (`packages/cli/src/ui/contexts/KeypressContext.tsx`), +6 / −1
- **Verdict:** `merge-after-nits`

## What it does

Tightens one line in the `emitKeys` generator at
`KeypressContext.tsx:655-666`. Before:

```ts
} else if (ch === '\b' || ch === '\x7f') {
  name = 'backspace';
  alt = escaped;
}
```

After:

```ts
} else if (ch === '\b' || ch === '\x7f') {
  name = 'backspace';
  alt = escaped && ch === '\b';
}
```

The fix targets a real Windows bug: Git Bash and MSYS2 send plain
Backspace as `ESC + DEL` (`\x1b\x7f`) instead of the standard `\x08`.
Under the old logic, the leading `ESC` set `escaped = true`, which set
`alt = true` for the trailing `\x7f`, which caused the keypress to be
dispatched as `Alt+Backspace` and trigger `DELETE_WORD_BACKWARD`. After
the fix, `alt` is only set to true when the trailer is actually `\b`
(`\x08`) — which is what genuine Alt+Backspace sends on every other
terminal. `\x7f` after `ESC` is reinterpreted as plain Backspace.

## Why it's correct

This is the right-shaped fix at the right layer. The keypress decoder
is exactly where terminal-quirk normalization belongs — pushing this
into a higher layer (e.g. action mapping) would force every consumer to
re-derive the Windows-bash special case. The author's claim "On
Windows terminals (Git Bash, MSYS2), pressing Backspace sends ESC + DEL
(0x1b 0x7f)" matches what readline and ConPTY observers have documented
for years; this is a well-known quirk.

The asymmetry the author preserves is also correct: genuine
Alt+Backspace on Linux/macOS does send `ESC + \b`, so the post-fix
predicate `escaped && ch === '\b'` correctly fires `alt=true` only for
the genuine case and leaves the Windows-bash backspace as plain.

## What's good

- Five-line diff for a multi-month UX paper-cut. No architecture
  changes, no dependencies, no API surface.
- The inline comment at lines 658-662 explains both the *what*
  (`\x1b\x7f` from Git Bash/MSYS2) and the *why* (would trigger
  DELETE_WORD_BACKWARD instead of DELETE_CHAR_LEFT) — future
  contributors don't need to reverse-engineer the asymmetric predicate.
- Targets a context (`KeypressContext.tsx`) that already owns
  cross-platform key normalization, not a new utility.

## Nits / risks

1. **No regression test.** The current file probably has an
   `emitKeys.test.ts` neighbor that verifies escape-sequence parsing.
   A test case that drives `\x1b\x7f` through `emitKeys` and asserts
   the result is `{ name: 'backspace', alt: false }`, plus a sibling
   case for `\x1b\x08` asserting `{ name: 'backspace', alt: true }`,
   would lock the asymmetric contract. Without the test, the next
   contributor who tries to "simplify" this branch will reintroduce
   the bug.
2. **Forwarded edge case.** What about `\x1b\x1b\x7f`
   (double-escape, e.g. Alt+Esc+Backspace)? The current `escaped`
   variable is set on the first ESC and presumably not cleared by a
   second ESC; if so, the trailer is still `\x7f` and `alt` will
   resolve to false. That's probably correct behavior (the user
   pressed Alt+Esc then plain Backspace) but it's untested. A
   doc-comment one-liner clarifying "we use *literal* `\b` as the
   tell that the user typed Alt+Backspace, because Windows-bash never
   sends `\b` for plain Backspace" would lock the invariant.
3. **`ConPTY` modern Windows Terminal** sends `\x7f` for plain
   Backspace too (no leading ESC). That case never had the bug
   because `escaped = false`, so `alt = false && true = false`. The
   author should mention in the PR body that the fix is *only*
   needed for Git Bash / MSYS2 — modern Windows Terminal users were
   never affected — to set expectations for which bug reports this
   closes.
4. **No issue link.** Almost certainly there's an existing report for
   "backspace deletes whole word in Git Bash" — link it so the closer
   knows what to mark fixed.
5. The `else if (ch === '\b' || ch === '\x7f')` branch is now
   semantically asymmetric internally (the two characters are *not*
   interchangeable for `alt`-determination). Consider refactoring to
   the explicit form:

   ```ts
   } else if (ch === '\b') {
     name = 'backspace';
     alt = escaped;            // genuine Alt+Backspace on Linux/macOS
   } else if (ch === '\x7f') {
     name = 'backspace';
     alt = false;              // \x7f is always plain Backspace; ESC prefix is Git Bash quirk
   }
   ```

   Reads more clearly than the combined predicate and matches the
   actual decision tree. Optional, but worth considering.

## Verdict

`merge-after-nits` — correct, minimal, and well-explained fix for a
known Windows terminal quirk. Add the `emitKeys.test.ts` regression
case before merging so the contract survives the next refactor pass,
and consider the explicit-branch rewrite for readability.
