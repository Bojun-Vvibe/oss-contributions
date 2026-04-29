# QwenLM/qwen-code#3749 — fix(cli): stop double-wrapping and double-printing API errors in non-interactive mode

- PR: https://github.com/QwenLM/qwen-code/pull/3749
- HEAD: `128430d`
- Author: umut-polat
- Files changed: 7 (+239 / -12)

## Summary

A 4xx error event in non-interactive mode flowed through both the
stream-error handler *and* the top-level `handleError`, each of which
called `parseAndFormatApiError` once — producing
`"[API Error: [API Error: 402 ...]]"` on stderr along with a duplicate
emission via the `JsonOutputAdapter.emitResult` path in TEXT mode. The
fix introduces an `AlreadyReportedError` marker that the runner throws
after it has already formatted-and-printed the error; the top-level
`handleError` and `JsonOutputAdapter` skip their own format/print steps
when they see it. Belt-and-suspenders: `parseAndFormatApiError` itself
gains an idempotency guard (`isAlreadyFormatted`) so even an unmarked
re-pass returns the input unchanged.

## Cited hunks

- `packages/cli/src/utils/errors.ts:22-46` — new `AlreadyReportedError`
  class with `isAlreadyReported = true as const` discriminator (so
  consumers can duck-type without `instanceof` across module-boundary
  identity issues — common pitfall when the class is bundled twice
  with different identities) and an `exitCode` field defaulting to 1.
- `packages/cli/src/utils/errors.ts:155-172` — new short-circuit at
  the top of `handleError`: if `isAlreadyReported(error)`, JSON mode
  still emits the structured payload exactly once (machine consumers
  shouldn't lose the error) but TEXT mode runs cleanup and re-throws
  without reformatting or reprinting. The asymmetry is correct —
  stderr is the duplicate-risk channel; the JSON adapter is the
  primary output channel for JSON mode and shouldn't be skipped.
- `packages/cli/src/nonInteractiveCli.ts:392-396, 564-569` — both
  stream-error sites now throw `AlreadyReportedError(errorText)`
  instead of `Error(errorText)`. The exit code stays 1 — same as
  before, with the marker being the only behaviour delta.
- `packages/cli/src/nonInteractiveCli.ts:768-789` — the catch-block
  emit path now checks `isAlreadyReported` via duck-typing
  (`(error as { isAlreadyReported?: boolean }).isAlreadyReported ===
  true`) — same defensive pattern. In TEXT mode, `skipAdapterEmit` is
  true and `adapter.emitResult` is bypassed; JSON / STREAM_JSON modes
  emit normally. Comment is clear about *why*: the adapter writes
  `errorMessage` to stderr in TEXT mode, which is the duplicate.
- `packages/cli/index.ts:96-103` — top-level `main().catch` adds an
  `AlreadyReportedError` short-circuit that bypasses the
  "An unexpected critical error occurred:" + stack-trace framing.
  Correct: a 4xx from the upstream API is a routine runtime failure,
  not a programmer-level bug, and dumping a stack trace mis-frames it.
- `packages/core/src/utils/errorParsing.ts:29-33, 65-71, 79-82` — the
  idempotency net: `API_ERROR_PREFIX` constant + `isAlreadyFormatted`
  helper check `value.startsWith("[API Error: ") &&
  value.trimEnd().endsWith("]")`. Both the `error.message` path (Error
  branch) and the plain-string path get the early-return guard.
  Crucially the **trailing-`]` check** prevents false positives where
  a raw upstream message happens to mention `[API Error: 502]`
  mid-string — that case still gets wrapped, as the new test at
  `errorParsing.test.ts:317-321` asserts.
- `packages/cli/src/utils/errors.test.ts:206-220` — regression test
  asserts `handleError(reported, mockConfig)` re-throws the same
  marker error *without* writing to stderr (`expect(mockWriteStderrLine)
  .not.toHaveBeenCalled()`). This is the load-bearing assertion: if the
  short-circuit ever regresses, this test catches it.
- `packages/cli/src/nonInteractiveCli.test.ts:867-928` — regression
  test mocks a 402 stream-error event, runs the full
  `runNonInteractive`, captures stderr, and asserts
  `"[API Error: [API Error:"` *never* appears anywhere in the
  collected output, plus that any line starting with `"[API Error: "`
  contains exactly one occurrence of that prefix. The defensive
  `dump`-on-failure pattern is excellent — when this test breaks in a
  year, the failure message will literally show the offending bytes.
- `packages/core/src/utils/errorParsing.test.ts:299-321` — three new
  idempotency tests cover (1) plain-string already-formatted,
  (2) `StructuredError.message` already-formatted, (3) raw message
  with prefix mid-string still gets wrapped. The third test is the
  one that locks the trailing-`]` anchor.

## Risks

- The duck-typed `isAlreadyReported` check at `nonInteractiveCli.ts:782-784`
  reaches into a runtime field. If a third party throws an Error that
  happens to have an `isAlreadyReported = true` property (e.g. a
  middleware that uses the same conventional name), the runner will
  silently drop that error's adapter emission. Vanishingly unlikely
  collision, but a more namespaced field
  (`isQwenCodeAlreadyReported = true` or a Symbol property) would close
  it.
- The `isAlreadyFormatted` guard makes `parseAndFormatApiError`
  idempotent for strings starting with `[API Error: ` and ending with
  `]`. Other formatters in the codebase (e.g. `[Tool Error: ...]`,
  `[Auth Error: ...]`, if any) don't get the same treatment; if
  they're chained through `parseAndFormatApiError` the double-wrap can
  return as `[API Error: [Tool Error: ...]]`. A more general
  "is-this-already-bracket-wrapped" predicate would be more durable.
- `AlreadyReportedError` extends `Error` — its name is
  `'AlreadyReportedError'` and `instanceof Error` is true. Any `try {
  ... } catch (e) { /* assume e is recoverable */ }` block elsewhere
  that catches `Error` and swallows will silently drop the marker, and
  the exit code will be lost. Worth a quick `git grep -nE
  'catch\s*\(\s*[^)]*Error'` audit to confirm no major call site does
  that.
- The fix doesn't address the *interactive* (TUI) double-format risk
  — only non-interactive mode is exercised. If interactive mode has the
  same double-pass shape via a different error path, it'll regress.
  Worth a follow-up audit to confirm the interactive runner uses a
  single-formatter convention.
- `JsonFormatter` is constructed inside the `handleError` short-circuit
  at `:160` for every JSON-mode marker error. Cheap, but if there's a
  shared formatter elsewhere it'd be more consistent to reuse it.

## Verdict

**merge-as-is**

## Recommendation

Land it; this is the canonical fix shape for double-format bugs (mark
the throw + idempotency net at the formatter), with regression tests
that lock both layers of defence independently. The Symbol-property
namespace nit can be a follow-up.
