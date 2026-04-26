# QwenLM/qwen-code #3630 — fix(telemetry): use safeJsonStringify in FileExporter to avoid circular reference crash

- **Author:** wenshao
- **Head SHA:** `0b8301e289de261f1770ed265dd08ccd27536bfe`
- **Size:** +52 / -1 (2 files)
- **URL:** https://github.com/QwenLM/qwen-code/pull/3630

## Summary

`FileSpanExporter.serialize` in
`packages/core/src/telemetry/file-exporters.ts:29` was calling
`JSON.stringify(data, null, 2)` directly on OTel `ReadableSpan`
instances. The spans hold a back-reference path
`_spanProcessor → BatchSpanProcessor → _shutdownOnce → BindOnceFuture →
_that → BatchSpanProcessor`, which forms a cycle and triggers
`TypeError: Converting circular structure to JSON` on every export. With
`DiagConsoleLogger` enabled the error gets re-emitted on stderr per
span, polluting the Ink TUI on every turn when `--telemetry-outfile`
is configured.

## Specific findings

- **The fix is the smallest correct one.** `file-exporters.ts:30` switches
  to `safeJsonStringify(data, 2)` — the project already has this util
  (`packages/core/src/utils/safeJsonStringify.ts`) for exactly this
  shape of cycle, so the change keeps a single canonical implementation
  rather than reintroducing a second sanitiser. Per the PR body this
  also matches the upstream gemini-cli fix, so future merges stay clean.

- **Regression test is well-targeted.**
  `packages/core/src/telemetry/file-exporters.test.ts:38-49` reproduces
  the exact `BatchSpanProcessor → BindOnceFuture` shape (`proc._shutdownOnce
  = future; future._that = proc; span._spanProcessor = proc`) and asserts
  three things: no throw, `"name": "span-1"` is preserved, and the cycle
  becomes `"[Circular]"`. That's the right granularity — broader cycle
  semantics live in `safeJsonStringify.test.ts`, this test only locks in
  *that the exporter delegates*.

- **One nit on the test fixture.** Casting `(exporter as unknown as
  SerializeAccess)` to reach a `protected` method works, but a cleaner
  alternative is to drive the public `export(spans, cb)` path with a
  fake `ReadableSpan` and assert on the file contents — that would also
  catch a future regression where someone calls `JSON.stringify` on a
  different code path (e.g. `MetricExporter.serialize`, which is the
  same class hierarchy and has the same hazard). Not blocking.

- **`MetricExporter` left on the old path.** The diff only touches the
  shared base `FileExporter.serialize`, so both span and metric exporters
  pick up the fix — confirmed by reading the unchanged inheritance chain.
  Worth a one-line note in the PR body to head off "what about metrics"
  review questions.

## Verdict

`merge-as-is`
