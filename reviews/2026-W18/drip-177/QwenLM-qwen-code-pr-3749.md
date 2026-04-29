# QwenLM/qwen-code#3749 — fix(cli): stop double-wrapping and double-printing API errors in non-interactive mode

- **Repo:** QwenLM/qwen-code
- **PR:** [#3749](https://github.com/QwenLM/qwen-code/pull/3749)
- **Head SHA:** `128430de06d4e82ddda586d0b9929b759e606639`
- **Author:** umut-polat
- **Size:** +239 / -12 across 7 files

## Summary

Fixes the non-interactive (`-p`) error pipeline. A routine 4xx from an
upstream API used to print three lines (the second of which was
`[API Error: [API Error: 402 ...]]`) followed by an "An unexpected critical
error occurred:" stack trace. The fix introduces an `AlreadyReportedError`
sentinel that says "I'm already on stderr, just exit non-zero — don't
reformat me, don't reprint me, don't surface me as a programmer-bug crash."

## What's actually going on

The bug had three independently-broken sites all formatting/printing the
same upstream error:

1. **The stream-error handler** in `runNonInteractive` formatted the
   incoming API error via `parseAndFormatApiError` and wrote it to stderr,
   then `throw new Error(errorText)` — which carried the *already-formatted*
   text up the stack as if it were a fresh raw error message.
2. **The top-level `handleError`** received that thrown `Error`, called
   `parseAndFormatApiError(error.message, ...)` again — which re-applied
   the `[API Error: ...]` prefix to a string that already had it, producing
   `[API Error: [API Error: 402 ...]]`.
3. **The `main().catch()` in `cli/index.ts`** then treated the (still-thrown)
   error as an unhandled programmer bug, printed
   `An unexpected critical error occurred:` plus a stack trace.

The fix introduces `AlreadyReportedError` (`packages/cli/src/utils/errors.ts`,
not in the visible diff but referenced from the import at `nonInteractiveCli.ts:39`)
with a contract: "the user-facing message is already on the wire, don't
touch it, just propagate the exit code." Three sites get patched:

- `packages/cli/src/nonInteractiveCli.ts:395` — first stream loop:
  `throw new Error(errorText)` → `throw new AlreadyReportedError(errorText)`.
- `packages/cli/src/nonInteractiveCli.ts:570` — second stream loop (turn-loop
  retry path): same swap.
- `packages/cli/src/nonInteractiveCli.ts:781-790` — `adapter.emitResult` is
  now skipped in TEXT mode when `error.isAlreadyReported === true`. JSON /
  STREAM_JSON modes still emit (per the comment at :784, the adapter is the
  *primary* output channel there, not a duplicate of stderr).
- `packages/cli/index.ts:96-104` — the top-level `main().catch` short-circuits
  on `AlreadyReportedError` and exits with the carried code instead of
  printing the "unexpected critical error" frame.

## Specific line refs

- `packages/cli/index.ts:17` — `import { AlreadyReportedError } from './src/utils/errors.js';`
- `packages/cli/index.ts:96-104` — the new short-circuit branch in
  `main().catch`. Reads cleanly: handles the FatalError case (existing),
  then the AlreadyReportedError case (new), then falls through to the
  programmer-bug "unexpected critical error" frame for everything else.
  Order matters: AlreadyReportedError must be checked before the generic
  Error frame, which it is.
- `packages/cli/src/nonInteractiveCli.ts:395` — first throw site, patched.
- `packages/cli/src/nonInteractiveCli.ts:570` — second throw site, patched.
  Comment at :567-569 explicitly cross-references the first site —
  good defense against a future refactor that "fixes" only one of the two.
- `packages/cli/src/nonInteractiveCli.ts:781-790` — the
  `isAlreadyReportedError` check uses a duck-typed
  `(error as { isAlreadyReported?: boolean }).isAlreadyReported === true`
  rather than `instanceof AlreadyReportedError`. Defensive against
  cross-realm prototype mismatches (multiple module copies, transpiler
  artifacts) but loses TypeScript-level guarantees. The `instanceof` check
  in `cli/index.ts:103` is the strict version — pick one and document why
  the relaxed check at `nonInteractiveCli.ts:786` is safer at that site.
- `packages/cli/src/nonInteractiveCli.test.ts:867-927` — new regression test
  asserting the double-wrap pattern `[API Error: [API Error:` never appears
  on stderr, and that any line beginning with `[API Error: ` contains the
  prefix exactly once via
  `expect(line.match(/\[API Error: /g)?.length ?? 0).toBe(1)`. The
  per-line assertion is the right bar — a global stderr-match could pass
  by accident if the test fixture changes.
- `packages/cli/src/utils/errors.test.ts:206-` — the "does not reformat or
  reprint AlreadyReportedError" test (truncated in the head, but the
  surrounding context shows it's the right surface to nail down the
  `handleError` short-circuit).

## Reasoning

This is a clean, correct fix to a real and embarrassing UX bug. The pattern
of "introduce a sentinel error type that carries a 'do not reformat' bit"
is the standard way to handle "multiple layers all want to format errors
at the boundary" in CLIs — both Node's own pattern (`error.code`
discriminator), Rust's `anyhow::Error` chain, and Python's traceback
suppression all converge on the same shape.

A few observations:

1. **Duck-typed vs `instanceof` mismatch.** `cli/index.ts:103` uses
   `error instanceof AlreadyReportedError` (strict), while
   `nonInteractiveCli.ts:786` uses
   `(error as { isAlreadyReported?: boolean }).isAlreadyReported === true`
   (relaxed). This is intentional — bundlers/transpilers can produce
   multiple `AlreadyReportedError` constructors and the `nonInteractiveCli`
   site is the more bundler-exposed of the two — but a `// see comment` link
   between the two sites would help the next reader understand why the
   approaches differ. Today neither site documents the asymmetry.

2. **`isAlreadyReported` field is undocumented.** The duck-type check at
   `:786` reads a field `isAlreadyReported` that the visible diff doesn't
   show being set. Presumably the `AlreadyReportedError` constructor sets
   `this.isAlreadyReported = true` — confirm in the actual `errors.ts`
   file that the field is set, otherwise the relaxed check at `:786`
   silently never fires and you get the bug back through the JSON/STREAM_JSON
   path.

3. **Test asserts double-wrap absence but not single-wrap presence.** The
   regression test at `nonInteractiveCli.test.ts:867-927` correctly asserts
   the bad pattern is gone, but doesn't positively assert the good pattern
   is on stderr. A `expect(stderrOutput).toMatch(/^\[API Error: 402 .../)`
   would close the loop — otherwise a future regression that *suppresses*
   the error entirely (skip-everything bug) would pass this test.

4. **`handleError` is still called in the catch path** at the second site
   (after the `skipAdapterEmit` decision). That's correct in JSON mode
   (where the adapter is the primary channel and `handleError`'s job is
   only to record telemetry), but in TEXT mode `handleError` will see the
   `AlreadyReportedError` and presumably no-op. Worth confirming
   `handleError(AlreadyReportedError, ...)` actually no-ops in text mode
   — the new test at `errors.test.ts:206` is the one to check. If
   `handleError` re-prints, you get the bug back.

5. **Error-shape contract on the wire.** The PR only handles the API-error
   case. If a programmer-side bug genuinely *is* an unexpected crash
   (e.g. a `TypeError` in user-tool code), the existing
   `An unexpected critical error occurred:` frame is what should fire —
   confirm the test suite has at least one case asserting that path
   *still works*. Otherwise this fix could regress the "real crash"
   reporting.

## Verdict

**merge-after-nits** — confirm the `AlreadyReportedError` constructor
actually sets `this.isAlreadyReported = true` so the duck-typed check at
`nonInteractiveCli.ts:786` fires (the visible diff imports the class but
doesn't show its definition; if the field isn't set, the JSON/STREAM_JSON
adapter-skip path silently never engages and the bug recurs there); add a
positive assertion to the regression test at
`nonInteractiveCli.test.ts:867-927` that stderr *does* contain a single
`[API Error: 402 ...]` line so a future "suppress everything" bug doesn't
silently pass; reconcile the `instanceof` vs duck-type asymmetry between
`cli/index.ts:103` and `nonInteractiveCli.ts:786` with a one-line comment
at each site explaining why the local check is the right shape; add or
verify a test case that a *genuine* programmer-bug crash (non-API
TypeError, etc.) still produces the "An unexpected critical error
occurred:" frame so this fix doesn't accidentally swallow real bugs;
and confirm `handleError(AlreadyReportedError, ...)` no-ops in TEXT mode
rather than re-formatting.
