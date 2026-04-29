# Review: QwenLM/qwen-code#3736 — feat(cli): improve slash command discovery

- **PR:** https://github.com/QwenLM/qwen-code/pull/3736
- **Head SHA:** `11fd64f0fad4aec3bce2fdbc746daff9ebc2416b`
- **Author:** LaZzyMan
- **Diff size:** +2428 / -333 across many files
- **Verdict:** `merge-after-nits`

## What it does

Phase 3 of a multi-phase slash-command UX overhaul (Phase 1/2 already shipped command metadata, cross-mode filtering, and prompt-command model invocation). This PR aligns the user-visible slash-command experience with Claude Code's: source badges in completions, argument hints, alias-hit labels, session-scoped recently-used ranking, a fully redesigned tabbed `/help` (General/Commands/Custom Commands views), richer mid-input ghost text + valid-token highlighting, and ACP `available_commands_update` enriched with the new metadata.

The 768-line `docs/design/slash-command/phase3-technical-design.md` is the centerpiece — it documents the existing baseline (audit of current `SlashCommand` shape, what each loader fills in today, current state of mid-input ghost text and `/help`), then the proposed delta (extend `Suggestion` with `source`/`sourceLabel`/`sourceBadge`/`argumentHint`/`matchedAlias`/`supportedModes`/`modelInvocable`, add `services/commandMetadata.ts` shared display helpers, refactor `Help.tsx` into a tabbed dialog).

## Specific citations

- `packages/cli/src/ui/components/SuggestionsDisplay.tsx` — `Suggestion` interface gets 7 new optional fields. The PR description says these are populated only when `mode === 'slash'` to keep file-completion / reverse-search paths zero-cost. Verify `mode === 'slash'` gating actually exists in the diff (the design doc claims it but readers will need to grep `useSlashCompletion.ts` to confirm).
- `packages/cli/src/ui/components/Help.tsx` — full rewrite into tabbed General/Commands/Custom panes. This is the highest-risk surface in the PR because Help is the first UI a new user touches.
- `packages/cli/src/services/commandMetadata.ts` (new) — five shared helpers (`getCommandSourceBadge`, `getCommandSourceGroup`, `formatSupportedModes`, `getCommandDisplayName`, `getCommandSubcommandNames`). Right call to factor these out of loaders — the design doc explicitly notes "do NOT put display logic in loaders".
- `packages/cli/src/ui/hooks/useSlashCompletion.ts` — generates enriched suggestions and adds session-scoped recently-used ranking. The PR commits to NOT persisting recently-used to disk (session-scoped only) which avoids a privacy/migration footgun.
- `packages/cli/src/acp-integration/session/Session.ts` — exposes new metadata via `availableCommands`. The design doc claims backward compatibility: existing `name`/`description`/`input` fields unchanged, new metadata in `_meta` or compatible fields. This needs explicit verification — ACP clients that pin schema strictly will break if any required field changed shape.
- The design doc lists ~10 built-in commands that need `argumentHint` backfill (`/model`, `/approval-mode`, `/language`, `/export`, `/memory`, `/mcp`, `/stats`, `/docs`, `/doctor`). Confirm all 10 are actually edited in the diff.
- The Phase 3 doc explicitly excludes `/release-notes` and re-implementation of `/doctor` — good scope discipline.

## Nits to address before merge

1. **The design doc is in Chinese (zh-CN);** the rest of the repo's `docs/design/` directory should be checked for language consistency. If other phase docs are English, translate at least the section headings + key invariants. If other phase docs are Chinese, fine.
2. **2428/-333 in a single PR is too big to review confidently.** Split if possible: (a) `Suggestion` interface + `commandMetadata.ts` + `argumentHint` backfills, (b) `Help.tsx` tabbed rewrite, (c) ACP enrichment. Each is reviewable on its own; the bundle is not.
3. **Tabbed `/help` is the highest-regression-risk UI change.** Add snapshot tests for each tab (General/Commands/Custom Commands), plus a test for empty Custom Commands tab (when no user/project/extension commands are loaded), plus keyboard-navigation tests (Tab, arrows, Esc).
4. **Recently-used ranking needs a clear test for tie-breaking.** When two commands have the same `usedAt`, what's the order? Alphabetical? Insertion order? Document and test.
5. **Source badges for built-in commands** are described as optional ("can be omitted to reduce noise") at design doc §4.2 but the actual implementation choice should be pinned with a comment in `getCommandSourceBadge`. Otherwise a future contributor will "fix" the missing badge.
6. **`alias hit` display** — when a user types `/m` and matches `/model` (aliased as `/m`), the suggestion should show `/m → /model` or `/model (matched: /m)`. The design doc references this from Claude Code's `getBestCommandMatch` but the precise rendered shape needs a screenshot in the PR body.
7. **ACP backward compat claim needs a concrete test.** Add an integration test that decodes the new `availableCommands` payload using the OLD ACP client schema and asserts no required-field error.
8. **`recently used` is in-process state.** What happens with multi-window/multi-session ACP clients that share a CLI process? Document expected behavior (per-process? per-ACP-session?).
9. **No mention of i18n.** If the CLI ships in multiple languages, hardcoded English in `getCommandSourceBadge` ("[Skill]", "[User]", etc.) blocks translation. At minimum, run badges through the existing `t(...)` helper.

## Rationale

This is a substantial UX-quality investment with a thoughtful, audit-grounded design doc that explicitly defers to existing source over earlier phase plans (good engineering posture: "if Phase 1 docs disagree with current main, current main wins"). The execution scope (recently-used in-session-only, no `commandType` re-introduction, ACP backward-compat) is right. Real concerns: (a) PR size makes it nearly unreviewable as a single unit, (b) `Help.tsx` rewrite is the highest-blast-radius UI change and lacks per-tab snapshot tests in the diff, (c) ACP backward-compat is asserted but not test-pinned, (d) i18n-readiness of new badge strings unclear. Approve after splitting (or at minimum: snapshot tests on Help tabs + ACP backward-compat integration test + a concrete tie-breaking rule for recently-used ranking).