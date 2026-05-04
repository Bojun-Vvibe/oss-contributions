# Review: QwenLM/qwen-code #3663

- **Title:** fix(cli): harden TUI flicker and streaming output stability
- **Head SHA:** `2058c43c1c6ba5d7afea25df2741ecf2faeb48f3`
- **Scope:** +4336 / -221 across 30 files (very large)
- **Drip:** drip-339

## What changed

Major rework of the TUI streaming/render path to fix the "clear storm"
class of flicker bugs. Adds a 739-line design doc, three new integration
test scripts under `integration-tests/terminal-capture/` (~1950 lines),
and rewrites `ConversationMessages.tsx`, `markdownUtilities.ts`, the
spinner, and the OpenAI-style content converter. Includes a
shell-execution-service tweak.

## Specific observations

- `docs/design/tui-optimization/streaming-clear-storm.md` (+739 / -0):
  the design doc is excellent context but lives under `docs/design/`
  alongside no other docs (verify path convention vs. siblings under
  `docs/`). Consider linking it from `CONTRIBUTING.md` so future authors
  find it.
- `packages/cli/src/ui/components/messages/ConversationMessages.tsx`
  (+165 / -14): the bulk of behaviour change. With +177 lines of paired
  test additions in `ConversationMessages.test.tsx`, coverage looks
  proportionate. Spot-check the new memoization for `key` stability —
  the original flicker bug class is usually a key-collision symptom.
- `packages/cli/src/ui/utils/markdownUtilities.ts` (+171 / -65): big
  rewrite of the markdown chunker, paired with `markdownUtilities.test.ts`
  (+34). Recommend a fuzz-style test that feeds randomly chunked streams
  of the same final string and asserts a stable end render.
- `packages/core/src/core/openaiContentGenerator/converter.ts` (+204 / -3):
  large additions to the OpenAI-content-generator converter from a PR
  whose stated scope is "TUI flicker". This widens blast radius into the
  core provider path. PR should either split the converter changes or
  justify why they are required to land together.
- `integration-tests/terminal-capture/streaming-clear-storm.ts` (+845)
  and friends: heavy new integration tests are valuable, but adding
  ~2k lines of test infrastructure alongside the fix doubles review
  burden. Confirm CI runtime budget can absorb the new harness.
- `packages/core/src/services/shellExecutionService.ts` (+16 / -6): why
  is the shell execution service touched in a TUI-flicker PR? Needs an
  explicit note in the PR body.

## Risks

- Scope creep: TUI fix bundled with provider-converter and shell-exec
  changes makes bisecting future regressions hard.
- New integration harness adds CI cost.
- Without seeing the spinner change in detail, possible interaction with
  TTY resize handlers.

## Verdict

**needs-discussion** — split the converter and shell-exec changes into
their own PRs (or explicitly justify the bundling), confirm CI budget,
and add the bisect-friendly per-file rationale.
