# sst/opencode #25557 — fix(tui): surface non-stop finish reasons in assistant header

- URL: https://github.com/anomalyco/opencode/pull/25557
- Head SHA: `d7d0361f8e98785377af806ae25e79682d07f5a9`
- Author: truffle-dev
- Scope: +82 / -2 across 3 files
- Closes: #23928

## Summary

When a model response ends abnormally — `length`, `content-filter`, or `error` — the
TUI / transcript export silently shows the truncated text with no indication of
why it stopped. This PR mirrors the existing `interrupted` indicator pattern and
appends a label like `· truncated by length limit` to the assistant header.

## What I checked

- `packages/opencode/src/cli/cmd/tui/util/transcript.ts` — new exported
  `formatFinishReason(finish)` is a pure switch over four cases (`length`,
  `content-filter`, `error`, default `""`). No Locale-aware formatting; the
  strings are English-only, which matches the surrounding code in this file
  (e.g. `formatAssistantHeader` already builds an English string). Acceptable.
- `packages/opencode/src/cli/cmd/tui/routes/session/index.tsx:1438` — the new
  `<Show when={!props.message.error && finishLabel()}>` block is correctly
  guarded so it never double-renders alongside the existing `· interrupted`
  span (which only fires when `props.message.error?.name ===
  "MessageAbortedError"`). The two indicators are mutually exclusive by
  construction.
- The `final()` memo at line ~1357 already excludes `tool-calls` and `unknown`
  from being treated as final, so `formatFinishReason` is never invoked on
  those values in the TUI path. Tests confirm the empty-string return for both
  anyway. Good defense in depth.
- `transcript.test.ts` — 13 new cases covering all four mapped values, the
  metadata-disabled path (`includeMetadata=false`), and three "should be empty"
  cases (`stop`, `tool-calls`, `unknown`, `undefined`). Test matrix is complete.

## Nits

1. `transcript.ts:90` — the new ternary inside the template literal is getting
   long. Splitting `finishLabel ? \` · ${finishLabel}\` : ""` into a local
   variable would match the style used for `duration` two lines above.
2. The four hardcoded English strings are good candidates for the same i18n
   pass that already covers `Locale.titlecase` here. Not a blocker — this PR
   doesn't introduce new translation surface; it just doesn't fix existing.
3. `error` finish reason produces `"ended with error"` which can be ambiguous
   when `props.message.error` is also present (a non-`MessageAbortedError`
   error). The `<Show when={!props.message.error && ...}>` guard suppresses it
   in that case, but if the model emits `finish: "error"` with no error object
   set, the user sees `"ended with error"` with no further detail. Minor;
   probably worth a follow-up to surface the actual error message.

## Risk

Very low. Pure additive UI string. No protocol, no storage, no streaming
behavior changes. Fallback to `""` for unknown values means provider-specific
finish reasons won't break the header.

## Verdict

`merge-as-is`
