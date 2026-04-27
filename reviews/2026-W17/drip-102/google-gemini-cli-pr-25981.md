# google-gemini/gemini-cli PR #25981 — fix(cli): dismiss update banner when /clear runs

- Link: https://github.com/google-gemini/gemini-cli/pull/25981
- Head SHA: `a94274e6d292490b7afbabf8f44db9710a78b306`
- Size: +27 / -0 across 3 files

## Summary

Threads `setUpdateInfo` through `useSlashCommandProcessor` (parallel to the already-threaded `setBannerVisible`) and invokes `setUpdateInfo(null)` from inside the `ui.clear()` callback so that `/clear` and `/new` actually dismiss the "Gemini CLI update available!" banner. The banner used to render as a history item (cleared transitively by `clearItems()`), but PR #5488 / commit `885af07d` moved it into the live-area `<Notifications>` component driven by `updateInfo` state owned by `AppContainer`, which broke the implicit dismissal — this PR adds the explicit reset to match the new state ownership.

## Specific-line citations

- `slashCommandProcessor.ts:108-112` adds `setUpdateInfo: (info: UpdateObject | null) => void` to the hook signature, and `:222-227` invokes `setUpdateInfo(null)` inside the `ui.clear()` callback right after `setBannerVisible(false)` — same shape, same call site, no surprise.
- `slashCommandProcessor.ts:273-279` adds `setUpdateInfo` to the `useCallback` deps array (would be a stale-closure bug if missed).
- `AppContainer.tsx:1051-1054` passes `setUpdateInfo` (already declared in the `[updateInfo, setUpdateInfo] = useState(...)` higher up in `AppContainer`) into the hook — no new state plumbing required, just wiring.
- `slashCommandProcessor.test.tsx:289-310` adds a focused `describe('Update banner dismissal')` block that builds a clear command via `createTestCommand`, dispatches `/clear`, and pins `expect(mockSetUpdateInfo).toHaveBeenCalledWith(null)`. Test fixture at `:127` and hook-call site at `:224` both add the new mock — consistent.

## Verdict

**merge-as-is**

## Rationale

Surgical 27-line fix that restores prior behavior after a known refactor (commit `885af07d` is named in the body), threads through an existing setter rather than introducing new state, and the test pins exactly the regression. The `UpdateObject` type import via `import type` is the right choice here (avoids runtime cost). No deps array footgun, no new lifecycle, no banner-state duplication — the `updateInfo` field continues to be the single source of truth and `/clear` now resets it.

If anything, `/new` likely needs the same reset (the PR body mentions it) — worth confirming whether the existing `/new` path also routes through `ui.clear()` or has its own callback that needs the same treatment. If `/new` already delegates to `ui.clear()` then this PR covers both; if it has a separate path, follow-up needed. Not blocking.
