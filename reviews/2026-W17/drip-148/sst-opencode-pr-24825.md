# sst/opencode #24825 — fix(tui): handle Zed selection byte offsets

- PR: https://github.com/sst/opencode/pull/24825
- Head SHA: `baa374e3ae749a8bb31a9d56c379256b41bd3264`
- Author: kitlangton
- Files: `packages/opencode/src/cli/cmd/tui/context/editor-zed.ts` (+23/−3), `packages/opencode/test/cli/tui/editor-context.test.ts` (+112/−2)

## Observations

- Root cause is correctly diagnosed: Zed persists selection ranges in its SQLite state DB as **UTF-8 byte offsets**, but the previous code at `editor-zed.ts:48-49` (`Math.min(row.selection_start, row.selection_end)` etc.) passed those byte offsets straight into JavaScript's UTF-16 code-unit-indexed `text.slice()` and the line/column calculator. For ASCII files the two are byte-identical so the bug never surfaced; any non-ASCII codepoint before the selection silently shifted the resulting `start`/`end` to the wrong character (Cyrillic shifts by N bytes per char minus 1, emoji by up to 3 bytes per surrogate pair beyond the JS index space).
- Fix shape: introduce `utf8ByteOffsetToStringIndex` at `:163-178` that walks the string codepoint-by-codepoint with `text.codePointAt(index)`, accumulates the UTF-8 byte length via `utf8.encode(text.slice(index, nextIndex)).length`, and returns the JS string index when the running byte total reaches the requested offset. The `nextIndex = index + (codePoint > 0xffff ? 2 : 1)` correctly handles the BMP/SMP boundary (surrogate pairs occupy 2 JS code units), so a `😀` (4 UTF-8 bytes, 2 UTF-16 code units, 1 codepoint) is advanced as a single codepoint cell while consuming 4 bytes from the offset budget. This is the textbook conversion.
- The choice to call `utf8.encode(...).length` for each codepoint slice (rather than computing UTF-8 length from the codepoint value via the well-known thresholds — `<0x80`→1, `<0x800`→2, `<0x10000`→3, else 4) is a correctness/perf tradeoff that lands on the safe side. For long files with deep selections this is O(n) `TextEncoder` allocations, but the lone module-level `const utf8 = new TextEncoder()` at `:23` reuses the encoder so per-call allocation is small. Fine for the typical Zed-selection sizes; would matter only for multi-MB files. Worth a comment acknowledging the choice.
- `offsetToPosition` at `:160-162` is also correctly retrofitted — the function is exported and used independently of `resolveZedSelection` (e.g., for offset-only callers), so layering the conversion at the public-API boundary rather than at the internal `offsetsToSelection` helper is the right encapsulation. Internal callers that already have a string-indexed offset are unaffected.
- Test surface is solid: 4 new selection-level tests (`converts Zed UTF-8 byte offsets to string offsets`, `non-ASCII text inside the selected range`, `emoji before the selected range`, `reversed Zed byte offsets`) plus 2 new cells in the `offsetToPosition` test. The reversed-offsets test at `:182-203` is the load-bearing one — it pins that the `Math.min`/`Math.max` reorder happens **before** the UTF-8 conversion, not after, so a Zed DB row written with end < start (rare but observed) still resolves correctly. The Cyrillic cell with `ЖЖЖЖЖЖЖЖЖЖ` (10 chars × 2 bytes each = 20 bytes vs 10 JS code units) is the cleanest demonstration of the bug class.

## Risks / nits

- `utf8ByteOffsetToStringIndex` returns `text.length` if `byteOffset` is past EOF — this matches the existing `offsetToPosition("one\ntwo\nthree", 100)` behavior pinned at the test's `:73` (clamps to last position). Fine. But if the byte offset lands *mid-codepoint* (e.g., `byteOffset = 2` inside a 4-byte emoji whose first byte is at 0), the loop returns `nextIndex` of the codepoint that completes past it — which means the selection **expands** to include the partial codepoint. This is the only sensible recovery behavior, but worth a comment because Zed in theory shouldn't ever emit mid-codepoint offsets, and a future bug there would manifest as silent off-by-one selections rather than an explicit error.
- The `if (byteOffset <= 0) return 0` guard at `:164` short-circuits negative byte offsets to position 0. Safer to assert/narrow at the call site (Zed doesn't emit negatives) so a negative number isn't silently normalized.
- No test for files containing combining-mark sequences (e.g., `é` as `e` + `\u0301`). The fix is byte-correct but the resulting "character" position the LSP-shape consumer sees may not match user intuition about cursor placement. Probably out of scope, but worth a doc note about what counts as a "character" in the returned `{line, character}` shape.
- The PR title says "Zed selection byte offsets" — accurate but understates that `offsetToPosition` was also broken. The fix scope is **all Zed-sourced offsets**, not just selection ranges. Worth widening the title or noting in the body.

## Verdict: `merge-as-is`

**Rationale:** Right diagnosis, surgical fix at the public-API boundary, and test cells that pin each of the bug's variants (selection-before, selection-inside, emoji, reversed). The implementation choice (per-codepoint `TextEncoder.encode().length`) is conservative but readable, and reuses a single module-level encoder. Nits above are commentary-quality, not blockers — the behavior is correct for the inputs Zed actually emits.

Date: 2026-04-29
