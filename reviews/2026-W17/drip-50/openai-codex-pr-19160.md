---
pr: 19160
repo: openai/codex
sha: f2924bf70caf62c1e2390257324ee9749a28c6d7
verdict: needs-discussion
date: 2026-04-25
---

# openai/codex#19160 — Make apply_patch streaming parser stateful

- **URL**: https://github.com/openai/codex/pull/19160
- **Author**: akshaynathan

## Summary

Replaces the existing "reparse the entire patch text on every
delta" `parse_patch_streaming()` entry point with a new
`StreamingPatchParser` (in `codex-rs/apply-patch/src/streaming_parser.rs`)
that holds parser state across `feed(line)` calls. PR claims a
10–15× speedup on reasonably sized patches at n=1. The
`parse_patch_streaming()` function and its `ParseMode::Streaming`
variant inside `parser.rs` are deleted entirely (343 lines removed
from `parser.rs`, 473 added in the new module).

## Reviewable points

- `codex-rs/apply-patch/src/lib.rs:24` adds
  `pub use streaming_parser::StreamingPatchParser;` and removes
  `pub use parser::parse_patch_streaming;`. **This is a breaking
  change to the `apply-patch` crate's public API.** The old
  `parse_patch_streaming(&str) -> Result<ApplyPatchArgs, ParseError>`
  callable is gone. Any external consumer (the codex tools handler is
  one — see `codex-rs/core/src/tools/handlers/apply_patch.rs:3-15`,
  which the PR also touches with -15/+3) and any third-party crate
  pinning `apply-patch` will break. The PR description doesn't call
  this out. If `apply-patch` is meant to be an internal-only crate
  (it lives under `codex-rs/`), a SemVer note in the crate's
  CHANGELOG (or at least the PR body) would be appropriate.

- `codex-rs/apply-patch/src/parser.rs:34-42` — the marker constants
  go from `const` to `pub(crate) const` so the new
  `streaming_parser.rs` module can reference them. Reasonable. The
  cleanup also removes a small but genuine bit of behavior: the
  removed `parse_one_hunk(..., allow_incomplete: bool)` parameter and
  the `if allow_incomplete && remaining_lines[0] == "@" { break; }`
  branch (around old line 318). Verify the new
  `StreamingPatchParser` covers the "single `@` while waiting for
  `@@` context" partial-input case — the old streaming parser had to
  invent that workaround precisely because the marker for a context
  hunk arrives one char at a time. If the new parser doesn't handle
  it, you'll see a transient `ParseError` flap during streaming for
  any update-hunk patch.

- `codex-rs/apply-patch/src/parser.rs:329` — the error message for
  empty Update-File hunks gains `Path::new(path).display()`
  formatting. Cosmetic but useful on Windows where path separators
  in error messages are otherwise inconsistent.

- The deletion of `test_parse_patch_streaming` and
  `test_parse_patch_streaming_large_patch_by_character` (parser.rs
  lines ~447–605 in the old file) leaves the new
  `streaming_parser.rs` as the sole test surface for streaming
  semantics. The new file is +473 lines including tests, but the
  diff doesn't show them in my window — confirm those large-patch
  *and* per-character feed tests have been re-implemented against
  `StreamingPatchParser`, otherwise a real coverage regression has
  been introduced under the cover of "look at the speedup".

- `codex-rs/core/src/tools/handlers/apply_patch.rs` shrinks by
  -15/+3. That's the hot path that calls the streaming parser on
  every model-token batch. Worth a benchmark in CI: a 10–15× win on
  parsing time only matters if it shows up at the tool-handler
  level (it should, but verifying it forecloses "we made the parser
  faster but the wrapping logic still re-allocates the whole buffer
  per call" surprises).

## Rationale

This is the right architectural change — incremental parsing with
held state is strictly better than reparsing on every delta — but
the PR couples three things at once: (a) algorithm rewrite, (b)
public API rename / removal, (c) deletion of the largest existing
test surface for the streaming path. A reviewer needs to (i) confirm
the new tests in `streaming_parser.rs` cover at least the same set
of partial-input edge cases the old tests asserted (especially the
character-at-a-time feed and the `@`-followed-by-newline transient),
(ii) decide whether the API rename is acceptable for this crate's
consumers, and (iii) ideally see a checked-in benchmark to justify
the "10–15×" claim and lock it as a regression guard. None of those
are blocking changes per se, but answering them out-of-band before
merge is the difference between "good cleanup" and "good cleanup
that quietly broke an edge case nobody noticed for two weeks".

## What I learned

Token-by-token streaming parsers are a recurring pain point in agent
codebases: the "happy path" is parsed many times per turn, so any
O(n²) re-parse cost is multiplied by token count. But the rewrites
are dangerous because the *invariants* the old re-parse relied on
("every call sees a complete prefix of a syntactically-valid
document") have to be re-encoded as state machines, and a
state-machine bug is much harder to detect than a parser bug — the
parser's failure mode is "throws ParseError", the state machine's is
"silently emits a wrong intermediate event". Treat
streaming-parser rewrites with character-at-a-time fuzz tests, not
just whole-input regression tests.
