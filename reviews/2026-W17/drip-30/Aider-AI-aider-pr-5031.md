# Aider-AI/aider#5031 — fix(io): tool_output falls back to ASCII on UnicodeEncodeError (#5029)

- PR: https://github.com/Aider-AI/aider/pull/5031
- Author: MukundaKatta
- +40 / -1
- Base SHA: `f09d70659ae90a0d068c80c288cbb55f2d3c3755`
- Head SHA: `ca600014be8dde8f0f38c0f32d896a093c7862cf`
- State: OPEN

## Summary

Wraps the `self.console.print(*messages, style=style)` call in
`InputOutput.tool_output` with a `try / except UnicodeEncodeError`
that retries with ASCII-coerced (`encode("ascii",
errors="replace").decode("ascii")`) message bodies. The motivating
bug (issue #5029) is that on legacy Windows terminals running
non-UTF-8 code pages (cp1251 etc.), Rich's tree/box-drawing
characters like U+22EE crash `/map` output entirely. A second
fallback path already exists in `_tool_message`; this PR brings
`tool_output` to parity.

## Specific findings

- `aider/io.py:1012` (head SHA `ca600014`) — the `try` wraps only
  the `console.print(...)` call, not the surrounding style
  construction, which is correct: `RichStyle(**style)` cannot
  raise `UnicodeEncodeError`.
- `aider/io.py:1019` — the fallback list comprehension `[str(m.plain
  if isinstance(m, Text) else m).encode("ascii",
  errors="replace").decode("ascii") for m in messages]` correctly
  unwraps `rich.text.Text` objects to their `.plain` form before
  ASCII-coercing. Without that, `str(Text(...))` would emit the
  Rich repr (e.g. `Text('foo', style='cyan')`), which is wrong.
- The retry passes the same `style=style` to `console.print` —
  Rich will then apply ANSI escapes around the (now-ASCII) body.
  On a terminal so broken it can't encode U+22EE, ANSI escapes
  may also fail; if the second `print` raises again, the
  exception will propagate. Acceptable — this is a fallback,
  not a guarantee.
- `tests/basic/test_io.py:386` — the new `test_tool_output_unicode_fallback`
  patches `io.console.print` to raise `UnicodeEncodeError` once
  then succeed. The assertion `mock_print.call_count == 2`
  pins the retry-once contract; the body assertion
  (`'\u22ee' not in joined and '?' in joined`) confirms the
  ASCII coercion ran. Clean regression test.

## Risks

- The fallback uses `errors="replace"` which substitutes `?` for
  every non-ASCII char. A user whose project legitimately
  contains, say, Cyrillic file names will see `?????` in tool
  output on a cp1251 terminal — not great, but strictly better
  than a crash that loses the whole message. A future
  improvement would be `errors="backslashreplace"` (renders
  `\u22ee` literally), which is searchable in scrollback.
- If `style=style` itself contains non-ASCII content (it
  shouldn't, but Rich allows arbitrary objects), the second
  `console.print` could raise the same `UnicodeEncodeError`
  unhandled. Unlikely; not worth a second `try`.

## Verdict

`merge-as-is`

## Rationale

Targeted defensive fix with a matching regression test; mirrors
an existing fallback in `_tool_message` so the codebase stays
internally consistent. The `errors="replace"` policy is the
right default for a *fallback* path — the goal is "don't crash,"
not "render perfectly."

## What I learned

When a codebase has one defensive fallback (`_tool_message`'s
existing handler) and the bug report shows a sibling code path
without the same handler, that's a strong signal to replicate
the pattern rather than introduce a third approach. Consistency
beats local cleverness in I/O fallback paths.
