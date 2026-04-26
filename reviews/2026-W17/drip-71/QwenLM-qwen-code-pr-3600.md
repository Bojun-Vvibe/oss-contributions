# QwenLM/qwen-code PR #3600 — handle shell line continuations in command splitting

- **PR**: https://github.com/QwenLM/qwen-code/pull/3600
- **Author**: @Jerry2003826 (JerryLee)
- **Head SHA**: `0394d0deed0a914b7e1a4e522b48f5f76171d4d9`

## Summary

`packages/core/src/utils/shell-utils.ts:227-230` — `splitCommands`
walks the command string char-by-char to extract sub-commands joined
by `&&`, `||`, `;`, or newlines. The bug: when an LLM-generated
shell command uses POSIX line-continuation (`\` at end of line, then
LF), the splitter saw the literal backslash-newline as part of the
command text and emitted broken segments. As a downstream effect,
`getCommandRoots()` extracted incorrect command roots, and the
allow-list guard could either over- or under-permit.

Fix: prepend a special-case branch in the parser loop that consumes
exactly two characters (`\` followed by `\n`) outside of single
quotes:

```ts
if (!inSingleQuotes && char === '\\' && nextChar === '\n') {
    i += 2;
    continue;
}
```

Two regression tests in `shell-utils.test.ts:445-456`:

- `'cd project && \\\ngit add file.php && \\\ngit commit -m "feat"'`
  → `['cd', 'git', 'git']` (three correct command roots)
- `'echo SAFE \\\r\nrm -rf /'` → `['echo', 'rm']` (CRLF is *not*
  treated as continuation — the `\r` keeps `rm -rf /` as a separate
  command so the security-relevant `getCommandRoots` still surfaces
  it for permission checks)

## Verdict: `merge-after-nits`

Correct fix, with one of the two tests being especially important
(the CRLF one) because it asserts the *security invariant*: a
sneaky-looking `\<CR><LF>` sequence does not hide a destructive
follow-up command from the allow-list.

## Specific references

- `shell-utils.ts:227-230` — the new branch. Position matters: it's
  *before* the existing `char === '\\' && i < command.length - 1`
  branch which preserves the backslash-escape pair. Without the
  earlier branch, `\\\n` would fall into the generic escape branch
  and become a literal `\n` inside `currentCommand`, defeating the
  fix. Order is correct.
- `shell-utils.test.ts:445-450` — positive case.
- `shell-utils.test.ts:452-456` — the CRLF negative case. Excellent.

## Concerns

1. **Single-quote scoping.** The guard `!inSingleQuotes` correctly
   skips the special case inside `'...'` (where `\` is literal in
   POSIX). Verify `inDoubleQuotes` behavior: in real bash, `\<LF>`
   is *also* a line continuation inside `"..."`. Test:
   `echo "hello \\\nworld"` — does the splitter handle this? Add a
   test to lock in current behavior either way.

2. **Other whitespace.** A continuation can technically be `\ `
   (backslash-space) followed by newline in some interpretations
   (interactive shells strip trailing whitespace). Out of scope but
   worth a follow-up if seen in the wild.

3. **`getCommandRoots` consumer audit.** Verify all callers of
   `splitCommands` benefit from this fix, not just `getCommandRoots`.
   If any caller previously *depended* on the literal `\\<LF>`
   appearing in the split output (unlikely but possible for
   transcript display), they'll see a behavior change.

## Nits

- The two-line branch could move into a helper
  `consumeLineContinuation(command, i)` that returns the new index
  or `null`. Marginal readability gain.
- The PR title mentions "command splitting" but the test names use
  "chained commands" — consider aligning.
