# Review — Aider-AI/aider#4968: fix: require exact SEARCH/REPLACE divider

- **Repo:** Aider-AI/aider
- **PR:** [#4968](https://github.com/Aider-AI/aider/pull/4968)
- **Head SHA:** `6cfa5ac441e66e6a463429222cc779a1923ac2d0`
- **Size:** +20 / -1 across 2 files
- **Verdict:** `merge-as-is`

## Summary

Closes #1803. Tightens the `DIVIDER` regex in `aider/coders/editblock_coder.py:387` from `^={5,9}\s*$` to `^={7}\s*$`, so a `=======` marker is required exactly. Previously, `.rst` source files containing heading underlines like `========` (8 equals) inside a SEARCH block would be misparsed as the divider, corrupting the edit. Adds one regression test in `tests/basic/test_editblock.py` exercising an `.rst` heading inside a SEARCH/REPLACE block.

## Technical assessment

Right diagnosis, minimal fix. The original `{5,9}` quantifier was historical leniency for LLM output that occasionally emits 5–9 equals; tightening to exactly 7 matches the documented prompt format that every system prompt teaches the model. The risk surface is "models that previously emitted 6 or 8 equals will now fail to parse" — but those failures are *better* than silent corruption because the user sees an explicit "could not find divider" error and the model retries on the next turn. The aider prompt templates have asked for exactly 7 for years, so model compliance is already high.

The regression test at `tests/basic/test_editblock.py` correctly uses the realistic failure case (`.rst` content with `========` underlines), which is exactly the shape reported in #1803. Verifies the SEARCH block parses with the heading intact rather than splitting at the underline.

## Pre-merge considerations

None blocking. Two minor follow-ups for a separate PR:

1. **Document the breaking change.** A one-line CHANGELOG entry "edit blocks now require exactly 7 `=` characters as the divider" would help users debugging "my edit suddenly stopped applying" if their custom system prompt taught the model to emit a different count.
2. **Consider symmetric tightening for `HEAD` and `UPDATED` markers.** If the `<<<<<<< SEARCH` / `>>>>>>> REPLACE` markers have similar `{5,9}` leniency, the same `.rst` collision could theoretically happen with `<<<<<<<` headings (rare but possible in some doc conventions). Quick grep of `editblock_coder.py` would confirm.

## Verdict rationale

`merge-as-is`. Surgical fix to a real bug with a regression test. The breaking-surface concern is minimal because the prompt has always asked for 7.
