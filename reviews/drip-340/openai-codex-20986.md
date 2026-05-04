# Review: openai/codex #20986 — feat: add line offsets to memory read MCP

- **PR**: https://github.com/openai/codex/pull/20986
- **Author**: jif-oai
- **Base**: `main`
- **Head SHA**: `9ad9d52e31bf0da17e9c015e513fe3f396d48e27`
- **Size**: +113 / −5 across 5 files (`backend.rs`, `local.rs`, `local_tests.rs`, `schema.rs`, `server.rs`)

## Scope

Adds an optional `line_offset` argument (1-indexed, default `1`) to the memories MCP `read` tool so callers can resume reading a memory file from a known line without re-consuming earlier content. Threads the new field end-to-end: schema → request struct → backend → response struct → tests.

## Substantive review

### `codex-rs/memories/mcp/src/backend.rs` (+6)
- `ReadMemoryRequest` gains `pub line_offset: usize` (line ~48) and `ReadMemoryResponse` gains `pub start_line_number: usize` (line ~55).
- Two new error variants on `MemoriesBackendError` (line ~99): `InvalidLineOffset` ("line_offset must be a 1-indexed line number") and `LineOffsetExceedsFileLength`. Clear messages, distinct from `InvalidPath` / `NotFile`.
- **Note**: this is an additive but breaking change to the `ReadMemoryRequest` struct. Any external implementor of the `MemoriesBackend` trait (or in-process caller building `ReadMemoryRequest` literally) must add the field. With `non_exhaustive` not present, every test in this PR had to be updated; in-tree call sites all appear handled, but it would be worth a `#[non_exhaustive]` if external implementations are anticipated.

### `codex-rs/memories/mcp/src/local.rs` (+30 / −2)
- New early guard at line ~95: `if request.line_offset == 0 { return Err(InvalidLineOffset); }`. Good — explicitly rejects the 0 case rather than treating it as 1.
- The new helper `line_start_byte_offset(content, line_offset)` (line ~316) walks `char_indices()` looking for `\n` and returns the byte offset of the next character. Correctness review:
  - Handles `line_offset == 1` by returning `0` immediately. ✓
  - Counts `\n` only — Windows `\r\n` files will still seek to the byte after `\n`, which is correct for slicing.
  - Returns `LineOffsetExceedsFileLength` if the requested line is past EOF — but **a single-line file with no trailing newline and `line_offset = 2` will also error**, which is the right behaviour.
  - The slice `&original_content[start_byte..]` is safe because `start_byte` came from `char_indices()`, so it lands on a UTF-8 boundary.
  - One subtle thing: `truncated = content != content_from_offset` (line ~117) — comparing a possibly long `&str` to a possibly long `String`. Functionally correct, but on very large content this is a full-content equality check on every read. The previous version had the same shape, so not a regression; just noting.

### `codex-rs/memories/mcp/src/local_tests.rs` (+66)
- Existing `read_rejects_directory_and_returns_file_content` test updated to pass `line_offset: 1` and assert `start_line_number: 1`. ✓
- New `read_supports_line_offset` test covers the happy path on `"alpha\nbeta\ngamma\n"` with `line_offset: 2` returning `"beta\ngamma\n"`. Good.
- Additional test cases for `line_offset == 0` rejection and `line_offset` past EOF would strengthen this further — they appear partially in the diff tail; please ensure both error variants have explicit assertions.

### `codex-rs/memories/mcp/src/schema.rs` (+4 / −2) and `server.rs` (+7 / −1)
- Schema adds `line_offset` as an optional integer property; `server.rs` deserialises and defaults to `1`. Standard MCP plumbing.

## Verdict

**merge-as-is** — the change is small, well-scoped, well-tested, and the helper function is correct. The only optional improvements are (a) `#[non_exhaustive]` on `ReadMemoryRequest` if external implementors are expected, and (b) explicit unit tests for both new error variants. Neither blocks merge.
