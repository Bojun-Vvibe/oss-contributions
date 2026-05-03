# QwenLM/qwen-code #3692 — fix(core): route countSessionMessages through parseLineTolerant

- **PR**: https://github.com/QwenLM/qwen-code/pull/3692
- **Head SHA**: `d4211c2592ef22e87da1dbdd51eefcfc2e569bc3`
- **Author**: qqqys
- **Size**: +33 / −7, 3 files

## Files changed

- `packages/core/src/services/sessionService.ts` (+8 / −5)
- `packages/core/src/utils/jsonl-utils.ts` (+3 / −2)
- `packages/core/src/utils/jsonl-utils.test.ts` (+22)

## Summary of change

`SessionService.countSessionMessages()` was using bare `JSON.parse(line)`
inside a try/catch that swallowed errors. When a session JSONL file has
the `}{`-glued corruption shape (referenced as bug #3606 in the diff
comment), the entire physical line was silently dropped, undercounting
messages and causing downstream UI misbehavior. The fix routes each line
through the existing `jsonl.parseLineTolerant<ChatRecord>(trimmed,
filePath)` helper which already understands how to split glued JSON
objects, and unifies the recovery code path with the rest of the JSONL
reader.

## Specific-line review

- `sessionService.ts:328-336` — replaces the try/catch+`JSON.parse` block
  with `for (const record of jsonl.parseLineTolerant<ChatRecord>(trimmed,
  filePath))`. The for-of correctly handles the case where one physical
  line yields multiple records — that is the *whole point* of this fix.
- `sessionService.ts:313-316` — the new doc comment explicitly cites
  `#3606` as the corruption shape. Future maintainers reading
  `countSessionMessages` will see why this is not a regular `JSON.parse`.
- `jsonl-utils.test.ts:85-104` — new `describe('parseLineTolerant', ...)`
  block with three cases:
  - happy path returns `[{a:1}]`
  - `{"uuid":"a"}{"uuid":"b"}` → `[{uuid:'a'},{uuid:'b'}]` — this is
    the regression-guarding test for #3606
  - `'not-json'` → `[]` — confirms graceful handling, no throw
  These three together exercise the contract `countSessionMessages`
  now relies on.
- `jsonl-utils.ts` (+3 / −2) — looks like a small export / signature
  tweak (the diff prefix shown is consistent with making
  `parseLineTolerant` newly exported and lightly refactored). Worth a
  maintainer eyeball to confirm no callers outside this PR depend on
  the old shape.

## Risk

Low. The only behavioral change is "lines that previously contributed 0
records may now contribute ≥ 1 record" — i.e. counts can only go up
for users with corrupted session files. Healthy session files produce
identical counts (the happy-path test guards this).

## Verdict

**merge-as-is** — well-targeted bug fix with three new tests that
directly cover the regression class. Reuses existing tolerant-parser
helper instead of inventing a parallel recovery path.
