# anomalyco/opencode #26013 — fix: navigate to home when --continue finds no sessions

- PR: https://github.com/anomalyco/opencode/pull/26013
- Head SHA: `7ce746318b1c4e357ff7cd92abfbef8a87327360`
- Base: `dev`
- Size: +4 / -0 across 1 file
- Files: `packages/opencode/src/cli/cmd/tui/app.tsx`

## Verdict
**merge-as-is**

## Rationale
Tiny, surgical fix for the exact failure mode in the linked issue (#25975). Without it, `--continue` on a project with zero sessions leaves the TUI sitting on the placeholder `sessionID: "dummy"` route, which the validator immediately rejects. The fix at `packages/opencode/src/cli/cmd/tui/app.tsx:382-385` adds the missing `else` branch to the lookup effect (around line 379): when no candidate session is found, it sets `continued = true` (so the effect doesn't keep re-running) and routes to `home`. That mirrors the existing CLI fallback in `run.ts` the author cites.

The `continued = true` assignment is important — without it, the effect dependency would re-fire and keep navigating home in a loop. Author got that right.

## Specific lines
- `packages/opencode/src/cli/cmd/tui/app.tsx:382-385` — new `else` branch is the entire change; logic is correct and matches existing patterns in the file.

## Nits (none worth blocking)
- Could add a unit test for the empty-sessions case, but the surface is a TUI effect and the project has no comparable tests for this path. Not worth gating.
