# sst/opencode PR #25368 — fix(transcript): wrap reasoning in `<thinking>` tags in markdown export

- **Head SHA**: `f2a4965ab0714f160d89a36c598559fc54636039`
- **Scope**: 2 files, +2/-2

## Summary

Replaces the `_Thinking:_` markdown prefix with `<thinking>...</thinking>` XML-style fences in the transcript exporter (`packages/opencode/src/cli/cmd/tui/util/transcript.ts:91`), with the matching test update at `packages/opencode/test/cli/tui/transcript.test.ts:152`.

## Notes

- Behavior change is purely formatting; the gate `options.thinking` is unchanged so disabled-thinking still emits `""`.
- `<thinking>` is the de-facto interop convention for several model families and is friendlier for downstream re-feeding into prompts than the italic "_Thinking:_" prefix, which most renderers don't visually distinguish anyway.
- One nit: there's no trailing newline inside the fence after `</thinking>` before the double-newline. Consistent with existing block conventions, fine as-is.
- No public API/schema impact; markdown export is a leaf utility.

## Verdict

`merge-as-is`
