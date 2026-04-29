---
pr: openai/codex#20186
sha: 781b6d3af268c8fb37c63d053bdc583abcc3374f
verdict: merge-as-is
reviewed_at: 2026-04-30T00:00:00Z
---

# nit: drop old memories things

URL: https://github.com/openai/codex/pull/20186
Files: `codex-rs/core/src/lib.rs`, `codex-rs/core/src/memory_trace.rs` (deleted),
`codex-rs/core/src/memory_trace_tests.rs` (deleted)
Diff: 0+/306-

## Context

Pure deletion PR. The `memory_trace` module hosting
`build_memories_from_trace_files` and `BuiltMemory` (an early offline
memory-summarization scaffold targeting `/v1/memories/trace_summarize`)
was superseded by the live memory pipeline and had no in-tree callers
remaining. Three lines of `pub use` re-exports at `lib.rs:198-200` were
the only public surface; the removal at `lib.rs:198-200` plus the file
deletions at `memory_trace.rs:1-230` and `memory_trace_tests.rs:1-73`
covers the complete change.

## What's good

- Zero-additions diff, three files touched, no behavioural change to
  any reachable code path — exactly the shape a "drop dead code" PR
  should have.
- The `pub use` removals at `lib.rs:198-200` are surgical: only the
  two re-exports for `BuiltMemory` and `build_memories_from_trace_files`
  go, leaving the surrounding `compact`, `memory_usage`, `otel_init`
  modules untouched.
- Deleting the test file alongside the module avoids the common
  follow-up "tests reference a deleted symbol → CI red" cleanup PR.

## Nits / follow-ups

- None. If anyone outside the workspace was importing
  `codex_core::BuiltMemory` they'll discover that at compile time on
  next bump, which is the appropriate breakage signal for an internal
  module that was never documented as public API.

## Verdict

`merge-as-is` — clean dead-code removal with co-located test cleanup,
no behavioural risk, the kind of PR that should land on green CI
without further discussion.
