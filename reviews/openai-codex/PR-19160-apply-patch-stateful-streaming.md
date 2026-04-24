# PR #19160 — make `apply_patch` streaming parser stateful

**Repo:** openai/codex • **Author:** akshaynathan • **Status:**
DRAFT • **Net:** +619 / −433

## Change

Replaces the existing `parse_patch_streaming(text) -> ApplyPatchArgs`
function with a stateful `StreamingPatchParser` that consumes deltas
via `push_delta(&str) -> Option<Vec<Hunk>>`. The parser owns a
line buffer and a `StreamingParserState` enum (`AwaitingHunk`,
`AddFile { path, contents }`, `UpdateFile { ... }`). Public
`parse_patch_streaming` and `ParseMode::Streaming` are removed; the
non-streaming `parse_patch` is routed through the same parser in
strict finalization mode. Net diff is large (+619/-433) because
old logic is being refactored, not just augmented.

## Load-bearing risk

The whole point of the previous `parse_patch_streaming` was that
it was *cheap and stateless* — every delta arrival re-parsed the
full accumulated buffer, which is O(n²) in worst case but trivially
correct. The new parser is O(n) per delta but holds mutable state
across calls. The risk surface this opens:

1. **Re-parse on resume / recovery.** If a streaming response is
   interrupted and replayed (network blip, stream cancel + retry),
   anyone holding a `StreamingPatchParser` instance now has half-
   parsed state. The old API made "throw it away and re-parse from
   the new full buffer" trivial; with the new API, callers must
   explicitly drop the parser and construct a new one, and any
   place that held a `&mut StreamingPatchParser` reference across
   await points needs auditing.

2. **`snapshot_if_changed()` defines progress-event semantics.**
   The `last_snapshot: Vec<Hunk>` field is what the parser uses to
   decide whether to emit a new progress event. The diff in the
   excerpt doesn't show `snapshot_if_changed`, but its equality
   check determines the emit cadence. If equality is by-value on
   `Vec<Hunk>` (`PartialEq` derive), every push of a single line
   to a chunk's `new_lines` triggers an emit — which may or may
   not be what consumers want. Old `parse_patch_streaming` callers
   could throttle externally; new API decides for them.

3. **`process_line` swallows errors silently.** The visible code
   sets `self.invalid = true` on a malformed `AwaitingHunk` line
   but doesn't surface why. After `invalid` is set, all subsequent
   `process_line` calls become no-ops. The caller can only
   discover this by getting `None` from `push_delta` forever — no
   typed error. The old `parse_patch_streaming` returned
   `Result<ApplyPatchArgs, ParseError>` with line numbers and
   diagnostic strings. This is a regression in observability.

4. **Removing `ParseMode::Streaming` from the public API is a
   breaking change** for any downstream that built tooling on top
   of `codex-apply-patch`. The PR description doesn't mention a
   migration note.

The DRAFT state suggests these are open questions. The +619/-433
churn also makes this a high-risk merge for any tree that has
in-flight changes against `parser.rs`.

## Concrete next step

Before un-drafting: (1) Document the `invalid` state as a typed
return on `push_delta`, e.g. `Result<Option<Vec<Hunk>>, ParseError>`,
so callers can log/recover. (2) Add a fuzz test (proptest) that
slices a corpus of valid patches at every byte boundary, feeds
each slice as its own `push_delta`, and asserts the final state
matches `parse_patch` of the full text. (3) Add a benchmark
comparison between the old re-parse-from-scratch and the new
stateful parser, on a representative patch — the refactor is
worth the complexity only if there's a measurable win on long
patches. (4) Decide and document the snapshot cadence
(per-delta vs. per-hunk-completion vs. per-line) and bench-test
the consumer impact.

## Verdict

Architecturally cleaner, but the API simplification has dropped
error visibility and added cross-call state coupling. Not safe to
land without typed errors and a fuzzing harness.

## What I learned

Stateful streaming parsers in Rust are a classic place where the
"obvious" cleanup (replace re-parse with incremental state) costs
you typed-error reporting unless you design for it from the start.
The `invalid: bool + return None forever` pattern is a smell —
caller can't distinguish "no progress yet" from "parse permanently
broken." Also: `+619/-433` on a parser is the cost of doing this
right; any reviewer who skims will miss the real concerns.
