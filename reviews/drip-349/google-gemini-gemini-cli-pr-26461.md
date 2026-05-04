# google-gemini/gemini-cli#26461 — fix(cli): prompt for editor selection on CTRL-X fallback

- **URL**: https://github.com/google-gemini/gemini-cli/pull/26461
- **Head SHA**: `5ea9c0e3c0c8`
- **Diffstat**: +269 / -3
- **Verdict**: `merge-after-nits`

## Summary

When the user invokes "open in external editor" (CTRL-X) and no editor is configured, this PR replaces the silent failure with a `RequestEditorSelection` event so the dialog can surface the picker, then resumes the edit once the user chooses. New unit tests cover the three branches (not-configured, success, unexpected error) for both `useTextBuffer` and `editorUtils`.

## Findings

- `packages/cli/src/ui/components/shared/text-buffer-external.test.ts:1-126` — clean, fully mocked tests using `renderHook` + `act`. Uses `vi.mock('node:fs', ...)` to stub the temp-file dance, which is the right level of isolation.
- The first test asserts `coreEvents.emit` is called with `CoreEvent.RequestEditorSelection` when `EditorNotConfiguredError` is thrown — exactly the user-visible contract this PR is adding.
- The third test expects "log feedback error for other errors" via `coreEvents.emitFeedback`. The body is truncated in this view but if it merely asserts the emit happens once, also assert the temp file is cleaned up (the original code path likely calls `unlinkSync`/`rmdirSync` in a `finally` — regression-test it now while you're touching this).
- `packages/cli/src/ui/utils/editorUtils.ts` defines the new `EditorNotConfiguredError` class. Make sure it's exported from the package index so downstream `instanceof` checks work in production builds, not just inside the test mock.

## Recommendation

Solid UX fix and the test layout matches the existing repo style. Confirm the cleanup-on-error path is asserted and that `EditorNotConfiguredError` is part of the public surface, then merge.
