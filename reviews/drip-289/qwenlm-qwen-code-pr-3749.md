# QwenLM/qwen-code PR #3749 — Stop double-wrapping and double-printing API errors in non-interactive mode

- Head SHA: `e3d3f3cb6882ccff78ef2f6a2b05d80dede4de06`
- URL: https://github.com/QwenLM/qwen-code/pull/3749
- Size: +308 / -27, 4 files (`packages/cli/index.ts`,
  `packages/cli/src/nonInteractiveCli.ts`,
  `packages/cli/src/nonInteractiveCli.test.ts`,
  `packages/cli/src/utils/errors.test.ts`)
- Verdict: **merge-as-is**

## What changes

Fixes a stderr regression where a 4xx (e.g. 402) from the upstream API
in non-interactive mode would be reported twice and *also* nested as
`[API Error: [API Error: 402 ...]]`. Two paths conspired:

1. The stream-error handler in `runNonInteractive`
   (nonInteractiveCli.ts:391-394 and 564-567) called
   `parseAndFormatApiError`, wrote it to `process.stderr`, then
   `throw new Error(errorText)` — wrapping an already-formatted
   string in a fresh `Error` whose `.message` is the formatted text.
2. The top-level catch in `index.ts` then ran the unhandled error
   through `handleError`, which re-applied formatting and re-emitted
   it under the "An unexpected critical error occurred:" preamble.

The fix introduces a sentinel `AlreadyReportedError` (errors.ts) thrown
by the stream-error handler instead of `Error`. The top-level
`main().catch` (index.ts:96-102) recognizes it and skips the "critical
error" preamble + stack dump, exiting with the captured `exitCode`
(default 1). The non-interactive runner also skips the
`adapter.emitResult` call in TEXT mode when `error instanceof
AlreadyReportedError` (nonInteractiveCli.ts:771-786) to avoid the
phantom blank/duplicate line.

## What looks good

- The new sentinel is *minimal* — `AlreadyReportedError extends Error`
  with an `exitCode` field. No new control-flow knobs, just a marker.
  This is the right shape for "I've already done my UX work, please
  don't redo it."
- The regression test `does not double-wrap or double-format an API
  error in non-interactive mode` (nonInteractiveCli.test.ts:867-919)
  asserts on the *exact* failure mode (`[API Error: [API Error:`
  substring at line 906) instead of stderr-line counts. The
  inline comment at lines 873-880 explains *why* counting writes
  would be wrong (JsonOutputAdapter legitimately emits the result
  message in TEXT mode for a separate concern). That's the right
  level of test rigor — pin the bug, not the line count.
- The "skip adapter.emitResult only in TEXT + AlreadyReported" gate
  (nonInteractiveCli.ts:778-785) is narrow. JSON / STREAM_JSON modes
  still emit normally because *there* the adapter is the primary
  output channel, not a duplicate of stderr. The asymmetry is
  correct and the in-line comment (lines 766-774) calls it out.
- exitCode preservation (index.ts:101) — `process.exit(error.exitCode)`
  uses the captured field, so a future caller that constructs
  `new AlreadyReportedError(msg, { exitCode: 2 })` won't have to
  re-plumb through `handleError`.

## Nits

1. The sentinel-class lives in `packages/cli/src/utils/errors.ts` but
   the equivalent "already-printed" idiom probably exists in other
   parts of the CLI (e.g. tool-error path, atCommand path). A quick
   `git grep "An unexpected critical error occurred"` to confirm
   nothing else writes that line *and* re-formats — and a JSDoc on
   `AlreadyReportedError` explicitly inviting other handlers to use
   it — would prevent the same bug class from sprouting in a sibling
   file.
2. The two stream-error handlers in `nonInteractiveCli.ts` (lines
   391-396 and 564-570) duplicate a 5-line block (`errorText =
   parseAndFormatApiError(...)`, `process.stderr.write(...)`,
   `throw new AlreadyReportedError(errorText)`). Worth extracting
   a `reportApiStreamError(error, config)` helper — non-blocking,
   but it's the kind of duplication that drifts.

## Risk

Low. The diff narrows existing throw sites from `Error` to
`AlreadyReportedError` (a subclass), so any caller that did
`catch (e: any)` and inspected `e.message` keeps working. The only
real behavior change is at the two sites that explicitly check
`instanceof AlreadyReportedError`, both of which are guarded and
preserve the legacy path for the `Error` case.

The regression test is precise enough that any future reformatter
that re-wraps the message will trip the `[API Error: [API Error:`
assertion immediately. Solid contribution.
