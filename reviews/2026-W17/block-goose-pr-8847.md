# block/goose #8847 — Add "Trimmed trailing whitespace" message to moim whitelist

- **Repo**: block/goose
- **PR**: #8847
- **Author**: spikewang (Spike Wang)
- **Head SHA**: 4ffe0a54b47c929f8512133159bbd89c98784993
- **Base**: main
- **Size**: +1 / −0 in `crates/goose/src/agents/moim.rs`.

## What it changes

Adds `"Trimmed trailing whitespace from assistant message"` to the
whitelist of expected/benign issue messages in `inject_moim`
(`moim.rs:39`). Previously the closure that builds `has_unexpected_issues`
only knew about three benign issues (consecutive-user merge,
consecutive-assistant merge, empty-tool-result placeholder); a new
message emitted by an upstream sanitizer was being misclassified as
"unexpected" and presumably triggering loud diagnostics.

## Strengths

- Smallest possible patch for the reported symptom. No behavior
  change beyond the whitelist update.
- The `&&` chain extension keeps the existing pattern intact;
  reviewers can see at a glance that the fix is additive.
- The message string presumably matches what some sanitizer/normalizer
  emits verbatim. (Substring match via `.contains` is correct given
  the upstream message may carry context.)

## Concerns / asks

- A whitelist-by-substring pattern is fragile: if the upstream
  message wording changes (e.g. "Trimmed trailing whitespace from
  assistant content"), this gate silently re-fires. Worth either
  (a) extracting the four whitelisted strings into named constants
  shared with the emitter, or (b) flipping the model to a structured
  enum/code rather than free-form strings. Out of scope for a 1-line
  hotfix but worth a follow-up tracking issue.
- No test added. A test that constructs an issue list including the
  new message and asserts `has_unexpected_issues == false` would
  lock in the fix. Given moim is centrally tested, this should be
  cheap.
- The existing whitelist entries use slightly different phrasings
  ("Merged consecutive user messages" vs. "Added placeholder to
  empty tool result"). Confirm the new entry's wording exactly
  matches what's emitted upstream — a typo here would silently
  fail to mask the issue.

## Verdict

`merge-as-is` — minimal, safe, additive. Asks (whitelist
constants, structured codes) belong in a follow-up.
