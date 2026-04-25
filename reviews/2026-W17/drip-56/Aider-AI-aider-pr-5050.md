# aider #5050 — fall back to ASCII in tool_output

- URL: https://github.com/Aider-AI/aider/pull/5050
- Head SHA: `590cd6e92a6dc2356b61303c14c54e0ce760d9a4`
- Verdict: **merge-as-is**

## What it does

Fixes #5029. `/map` prints the repository map through
`InputOutput.tool_output()`. On non-UTF-8 consoles (e.g. Windows cp1251),
Rich raises `UnicodeEncodeError` while writing the `⋮` separator and the
exception escapes — even though the sibling `_tool_message()` path
already had an ASCII-safe fallback. This PR brings `tool_output()` to
parity.

## Diff notes

- Patch in `aider/io.py:1009-1020`:
  ```python
  try:
      self.console.print(*messages, style=style)
  except UnicodeEncodeError:
      safe_messages = []
      for message in messages:
          if isinstance(message, Text):
              message = message.plain
          safe_messages.append(
              str(message).encode("ascii", errors="replace").decode("ascii"),
          )
      self.console.print(*safe_messages, style=style)
  ```
  Three things done right:
  1. The `Text` → `.plain` unwrap mirrors what `_tool_message()` does, so
     Rich-styled args don't get encoded as their `repr()`.
  2. `errors="replace"` substitutes `?` for unencodable chars, matching the
     existing helper's behavior — `⋮` becomes `?` not a crash.
  3. The retry still goes through `self.console.print(..., style=style)`,
     so the user keeps any color/bold from the original call rather than
     dropping to `print()`.
- Test in `tests/basic/test_io.py` mocks `console.print` with a
  `[UnicodeEncodeError(...), None]` side_effect list, which is exactly the
  right shape — first call raises, second call (the ASCII retry) succeeds,
  and the assertion checks the second call's args got `⋮ → ?` substituted.

## Risk surface

- The exception handler is narrow (`except UnicodeEncodeError`) and only
  re-runs `console.print` once. If the second call also raises, it
  propagates — that's correct, because a second-pass failure means
  something is wrong with the console, not with the input bytes.
- No behavior change on UTF-8 consoles: the `try` block hits the original
  `console.print` path on the happy path with no overhead.

## Why this verdict

Tiny, focused fix with a precise regression test, mirroring an existing
fallback pattern already in the file. Nothing to nit.
