# ollama/ollama#15705 — Prevent qwen3coder tag regex from matching across newlines

- PR: https://github.com/ollama/ollama/pull/15705
- Author: mverrilli (Michael Verrilli)
- +66 / -1
- Head SHA: `824f1768b26828eb0a9f937bfef41fba723db636`

## Summary

One-character regex fix for issue #15574. The `qwenTagRegex` was
`<(\w+)=([^>]+)>` and `[^>]+` matches newlines, so a parameter value
containing a `<word=` pattern (the issue report uses BBC BASIC code like
`IF C<1w=P-SQR(1-C)`) caused the regex to greedily span across lines and
consume the real `</parameter>` closing tag, until it finally hit the `>`
in `</function>`. The downstream `xml.Unmarshal` then failed with
`element <parameter> closed by </function>`. Fix is `[^>]+` → `[^>\r\n]+`,
restricting the regex to a single line.

## Specific findings

- `model/parsers/qwen3coder.go:395` (SHA
  `824f1768b26828eb0a9f937bfef41fba723db636`) — the new regex is
  `<(\w+)=([^>\r\n]+)>`. Including both `\r` and `\n` in the negated class
  is correct and slightly more robust than the PR description suggests
  (description says `\n`, code includes both — handles CRLF input
  conservatively). The companion comment on `qwenXMLTagRegex` at line 397
  notes that `[^"]` in that regex is safe because `xml.EscapeText` has
  already encoded literal newlines as `&#xA;` before the regex runs —
  good context for future readers.
- `model/parsers/qwen3coder_test.go:1124` —
  `TestQwen3CoderParserAngleBracketsInParameterValue` is the end-to-end
  regression test, verifying both that the parser produces exactly one
  tool call and that the `source` argument round-trips byte-for-byte
  including the literal `<` character. Good shape: tests the public API,
  not the regex internals.
- `model/parsers/qwen3coder_test.go:1213` — the
  `TestQwenXMLTransform` "angle brackets in parameter values do not
  corrupt closing tags" case asserts the intermediate XML transform output
  with `&lt;` escaping. Combined with the second case
  ("multiple angle-bracket patterns in same parameter value"), this pins
  down both single-occurrence and multi-occurrence behavior on the
  transform layer.

## Verdict

`merge-as-is`

## Rationale

Trivially correct: the negated character class `[^>\r\n]+` is the standard
fix for "regex unintentionally crosses lines," and Qwen3-coder's wire
format always puts each tag on its own line so the constraint cannot
generate false negatives. Test coverage is layered (transform-level XML
expectation + end-to-end parser+arg extraction), the failure mode mapped
in the PR description matches the issue report verbatim, and the
companion comment on `qwenXMLTagRegex` proactively documents why the
neighbor regex is safe. Ship.
