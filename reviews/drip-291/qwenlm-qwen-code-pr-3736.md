# QwenLM/qwen-code PR #3736 — feat(cli): improve slash command discovery

- URL: https://github.com/QwenLM/qwen-code/pull/3736
- Head SHA: `b11b29e6b2ca752d70df0aa2a40c9461b0e8e5bf`
- Author: LaZzyMan
- Repo: QwenLM/qwen-code

## Summary

Phase-3 of the slash-command UX rework: enriches suggestion items with
source/argumentHint/aliasHit metadata, refactors `Help` into a tabbed
panel, augments mid-input ghost text and slash-token highlighting, and
exposes additional metadata over ACP `available_commands_update`. Large
patch (~36 files, +cli/+core, design docs included).

## Specific notes

- `packages/cli/src/services/commandMetadata.ts` — new shared module
  centralizing `getCommandSourceBadge / getCommandSourceGroup /
  formatSupportedModes`. Right call: keeps Loaders free of UI logic, as
  the design doc at `docs/design/slash-command/phase3-technical-design.md`
  lines 145–152 explicitly argues. Verify this module has unit tests for
  the source-mapping table — the design doc enumerates 8 distinct badge
  outcomes and only some appear in the diff hunk.
- `packages/cli/src/ui/components/SuggestionsDisplay.tsx` — `Suggestion`
  is extended with `source`, `sourceLabel`, `sourceBadge`,
  `argumentHint`, `matchedAlias`, `supportedModes`, `modelInvocable`.
  All optional, so non-slash modes (file, reverse search) won't break.
  Confirm the `[Built-in]` badge is suppressed by default to avoid badge
  spam on `/help`, `/clear`, etc. — the design doc recommends it.
- `packages/cli/src/ui/components/Help.tsx` — refactor from list-dump to
  tabbed panel. This file is a long rewrite; double-check it still
  renders correctly under `useMinimalUi` / non-interactive shims.
- `packages/cli/src/acp-integration/session/Session.ts` — new metadata
  is placed in `_meta` per the design doc's "ACP backward compat"
  constraint (§1.2). Good. Confirm `name`, `description`, `input.hint`
  byte-for-byte unchanged for existing clients.
- `packages/cli/src/ui/hooks/useSlashCompletion.ts` — alias-hit
  preserved through the suggestion. Recently-used ordering is session
  scoped only (no disk persist), matching the design hard constraint.
  Watch for unbounded growth of the LRU map for sessions that page
  through many commands.
- `packages/core/src/tools/task-stop.test.ts` shows up in the file list
  but unrelated to slash command UX — likely an incidental fix; should
  be split out for a cleaner reviewable history.

PR is large but matches a published design doc (Phase 3) line-by-line.
Worth a maintainer pass to verify the badge-suppression rule and the
incidental `task-stop.test.ts` change.

verdict: merge-after-nits
