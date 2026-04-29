# PR #26148 — fix(ui): fix issue with box edges

- **Repo:** google-gemini/gemini-cli
- **Link:** https://github.com/google-gemini/gemini-cli/pull/26148
- **Author:** gundermanc
- **State:** DRAFT
- **Head SHA:** `9482f81d50ba8ff434befb1fb3c26a9a95d7d9d3`
- **Files:** `packages/cli/src/ui/components/messages/ToolGroupMessage.tsx` (+4/-2), `packages/cli/src/ui/components/messages/ToolGroupMessage.test.tsx` (+33/-0), `packages/cli/src/ui/components/messages/__snapshots__/ToolGroupMessage.test.tsx.snap` (+16/-0) — total +56/-2

## Context

`ToolGroupMessage` is the renderer that draws boxed UI for groups of tool calls in the TUI. Adjacent tool groups normally share a border seam — the bottom of group N is the top of group N+1, drawn once. The renderer decides whether to draw a top border for group N+1 by inspecting the *previous* group's identity:

```ts
const isFirstProp = !!(isFirst ? (borderTopOverride ?? true) : prevIsCompact);
```

I.e., "draw the top border if this is the first group, *or* if the previous group was a compact tool (one that doesn't draw its own bottom border)."

The bug: `update_topic` is rendered as a non-compact, *non-bordered* topic message (`Middle Topic` in the snapshot, no surrounding box). So when a real tool group followed an `update_topic`, the previous group "doesn't draw its own bottom border" *but isn't compact* — `prevIsCompact` was `false` and the top border was suppressed. The user sees the second tool's box missing its top edge.

## What changed

Symmetric one-line fix at *both* of the two render sites that compute `isFirstProp`:

1. `ToolGroupMessage.tsx:195-196` — first call site (the inline `groupedTools.map(...)` path):

   ```ts
   const prevIsTopic =
     prevGroup && !Array.isArray(prevGroup) && isTopicTool(prevGroup.name);
   ```

   And the predicate at line 232 changes from `prevIsCompact` to `prevIsCompact || prevIsTopic`.

2. `ToolGroupMessage.tsx:368-370` — second call site (the rendered-element path further down). Same `prevIsTopic` declaration, same `prevIsCompact || prevIsTopic` swap.

The duplication-across-call-sites is pre-existing — both sites were already reading `prevIsCompact` independently. The PR keeps both in sync, which is the right move (a unified helper would be a separate refactor).

The new test at `ToolGroupMessage.test.tsx:371-401` builds the exact failing topology — `read_file` ⇒ `update_topic` (middle) ⇒ `write_file` — and snapshots the rendered frame. The snapshot at `__snapshots__/ToolGroupMessage.test.tsx.snap:144-156` shows the correct shape: top box closes (first tool), `Middle Topic` renders unboxed, second box opens cleanly with intact `╭` corners on its top border.

## Design analysis

This is the right minimal fix.

1. **Symmetric application across both sites.** The renderer has two parallel paths that compute `isFirstProp` from `prevIsCompact`. The PR updates both. Without the second update, the bug would be fixed only on one render path (likely the more common one) and the other would still produce the missing border on whatever it renders. Symmetry is the right discipline here.

2. **`isTopicTool(prevGroup.name)` is the existing predicate** — same one already used to gate `TopicMessage` rendering. Using the canonical predicate (rather than `prevGroup.name === UPDATE_TOPIC_TOOL_NAME`) means future "topic-class" tools that get added (e.g. `set_subtopic`) automatically participate in the border-fix without further edits.

3. **Snapshot test is the right contract layer for this fix.** The bug is purely visual; pinning the rendered character grid is the only test that would actually catch a regression. The snapshot diff (3 tools, 16 new snapshot lines) is a reasonable size — not so big that visual review of future diffs becomes a chore.

## Risks

1. **`prevIsTopic` is the only "non-compact, non-bordered" group type currently considered.** If another bordered/non-bordered group type is added in the future (a "spinner" group, a "streaming-thought" group, etc.), it'll re-introduce a similar variant of this bug. A short comment near the new `prevIsTopic` declarations noting "extend this when introducing other unboxed prev-group types" would help future maintainers.

2. **Snapshot is fixed-width 76 cols.** The terminal width is implicit in the snapshot. A theme/locale that produces wider tool descriptions could force a re-snapshot. Pre-existing issue with snapshot tests in this file; not introduced by this PR.

3. **DRAFT state + only macOS validated.** The pre-merge checklist shows only macOS `npm run` checked. The change is purely string/component logic — no platform surface — so cross-platform validation is procedural. But the PR is in DRAFT, so the author isn't claiming readiness yet.

4. **Two parallel definitions of `prevIsTopic`.** Since both call sites of `prevIsCompact` already independently compute the same thing, adding `prevIsTopic` independently in both is consistent with the existing pattern. It's a smell that merits a future refactor (extract `getPrevGroupKind(prevGroup): { isCompact, isTopic, ... }`) but not in this PR's scope.

## Verdict

**Verdict:** merge-after-nits

Right diagnosis (an unboxed-but-non-compact prev-group type was the missing branch in the border predicate), right fix (symmetric application at both render sites using the canonical `isTopicTool` predicate), right test (snapshot pinning the rendered char grid).

Nits:
- Add a one-line comment above the new `prevIsTopic` declarations documenting "extend this when introducing other unboxed prev-group types" so future additions don't silently re-break the seam.
- Take the PR out of DRAFT once the author is satisfied the cross-platform check is procedural (it is).
- Optional follow-up: extract a `getPrevGroupKind(prevGroup)` helper to consolidate the two parallel call-site computations.

---

*Reviewed by drip-156.*
