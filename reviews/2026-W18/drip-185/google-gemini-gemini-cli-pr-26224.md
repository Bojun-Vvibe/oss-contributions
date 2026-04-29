---
pr: google-gemini/gemini-cli#26224
sha: 4203ef78c7fb2458ee1717319cd29be6e13e8b82
verdict: merge-as-is
reviewed_at: 2026-04-30T00:00:00Z
---

# fix(cli): use byte length instead of string length for readStdin size limits

URL: https://github.com/google-gemini/gemini-cli/pull/26224
Files: `packages/cli/src/utils/readStdin.ts`,
`packages/cli/src/utils/readStdin.test.ts`
Diff: 65+/4-

## Context

`readStdin` at `readStdin.ts:33-44` enforced `MAX_STDIN_SIZE = 8MB` by
comparing against JavaScript string length. JavaScript string `length`
counts UTF-16 code units (so a 3-byte UTF-8 char like `'한'` counts as
1, a 4-byte emoji like `'🌟'` counts as 2 surrogate-pair units), while
the actual byte cost when later UTF-8-encoded for transport is
typically 2-4x higher. Result: a Korean-language piped prompt could be
~24MB on the wire while passing the "8MB" guard. PR replaces every
length comparison with `Buffer.byteLength(chunk, 'utf8')` and adds a
boundary-safe truncation helper.

## What's good

- `truncateUtf8Bytes` at `readStdin.ts:11-18` is the textbook UTF-8
  boundary walk: hit the byte limit, then `while (end > 0 && (buf[end]
  & 0xc0) === 0x80) end--` walks backward over continuation bytes
  (`10xxxxxx`), leaving `end` at the lead byte of the incomplete
  sequence which is then *excluded* from the slice. Returns valid
  UTF-8 every time — never a `\uFFFD` replacement char in the output.
- `if (buf.length <= maxBytes) return str;` early-exit at `:13` avoids
  the round-trip cost on the common case (chunk under limit).
- Replacement of `chunk.length` with `Buffer.byteLength(chunk, 'utf8')`
  at `readStdin.ts:36,46` is correct and applied symmetrically — the
  size accumulator (`totalSize`) and the over-limit detection both
  use byte counts now, so they can't disagree.
- Two regression tests at `readStdin.test.ts:143-189` cover the two
  bug classes:
  - `should truncate multi-byte characters at byte boundary` —
    feeds `2796203 * '한'` (one over the 8MB / 3 boundary), asserts
    `resultBytes === Math.floor(MAX_STDIN_SIZE / 3) * 3` (clean
    multi-of-3 byte boundary), and crucially asserts
    `result.not.toContain('\uFFFD')` (no mid-character split).
  - `should use byte length instead of string length for limit` —
    feeds `8M * '한'` (~24MB on the wire), asserts the byte length
    is ≤ 8MB *and* `result.length < charCount` (truncation actually
    happened, didn't just pass through). The `result.length < charCount`
    assertion is the one that would catch a regression where someone
    accidentally re-introduces string-length comparison.
- The truncation message
  `"Warning: stdin input truncated to ${MAX_STDIN_SIZE} bytes."` at
  `:51` already said "bytes" — the existing message was honest about
  the unit, the implementation just wasn't honoring it. Fix aligns
  reality with the existing user-facing claim.

## Nits

- None blocking. Optional: extract `MAX_STDIN_SIZE` to a module-level
  constant alongside `truncateUtf8Bytes` so consumers (or future tests)
  can import and reference it instead of redefining `8 * 1024 * 1024`
  in each test.
- The helper handles the "incomplete UTF-8 at the end" case but
  doesn't have an explicit comment about the `(buf[end] & 0xc0) ===
  0x80` continuation-byte mask — fine, idiomatic, but a one-line
  link to RFC 3629 §3 would help the next maintainer.
- Consider a third test for the surrogate-pair (4-byte UTF-8 emoji)
  case to exercise the longer continuation-byte walk: `'🌟'.repeat(N)`
  where N straddles the boundary. Current tests cover the 3-byte
  Korean case but not the 4-byte case, and the `while` loop walks
  past 1 vs 3 continuation bytes are different code paths in
  principle.
- `Buffer.byteLength(chunk, 'utf8')` is called twice on the same chunk
  in the over-limit branch (once at `:46`, once implicit in the
  `Buffer.from` inside `truncateUtf8Bytes`). Negligible perf, but
  passing the precomputed length to `truncateUtf8Bytes` would let it
  skip the `buf.length <= maxBytes` early-exit recomputation.

## Verdict reasoning

Textbook UTF-8 byte-vs-character bug fix with a continuation-byte-
safe truncator and two regression tests covering both the
"multi-byte truncation produces valid UTF-8" and "byte length
actually enforces the byte limit" assertions. The
`expect(result).not.toContain('\uFFFD')` and `expect(result.length
< charCount)` are exactly the right fences. No API change, no
behaviour change for ASCII-only stdin, and the existing user-facing
"truncated to N bytes" message is now actually honest. Landable
as-is.
