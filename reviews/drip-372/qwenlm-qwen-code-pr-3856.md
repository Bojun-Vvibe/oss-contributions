# QwenLM/qwen-code#3856 — feat(cli): polish --add-dir / --include-directories feature

- **URL:** https://github.com/QwenLM/qwen-code/pull/3856
- **Head SHA:** `a0daf50c065f48f793c357dc3a600ca60d4672c9`
- **Files touched:** 10 (+413 / −3)
  - `.gitignore` (+3 / −1)
  - `packages/cli/src/config/config.ts` (+1 / −1)
  - `packages/cli/src/i18n/locales/en.js` (+11 / −0)
  - `packages/cli/src/ui/commands/directoryCommand.tsx` (+118 / −0)
  - `packages/cli/src/ui/commands/directoryCommand.test.tsx` (+156 / −0)
  - `packages/core/src/config/config.ts` (+6 / −0)
  - `packages/core/src/memory/relevanceSelector.ts` (+3 / −0)
  - `packages/core/src/memory/relevanceSelector.test.ts` (+49 / −1)
  - `packages/core/src/utils/workspaceContext.ts` (+12 / −?)
  - `packages/core/src/utils/workspaceContext.test.ts` (+54 / −0)

## Summary

Polishes the `--add-dir` / `--include-directories` workflow with four discrete improvements:
1. New `/directory remove` slash subcommand (with tab-completion that filters out initial directories) and persistence to workspace settings.
2. Startup `stderr` warning listing `--include-directories` paths that don't exist or aren't readable (previously silently debug-logged).
3. Updated `--add-dir` CLI help text documenting path resolution and skip behavior.
4. New `WorkspaceContext.getSkippedDirectories()` accessor backed by an internal tracker in `addDirectory()`.

The diff also bundles two arguably-unrelated changes that should be flagged separately (see Concerns).

## Line-level observations

- `packages/cli/src/ui/commands/directoryCommand.tsx:+118` — adds a `remove` subcommand alongside the existing `add` and `show`. Tab-completion logic correctly filters out the workspace's initial directories (so users can't try to remove the cwd or other startup-provided roots), and the action handler validates against missing-directory and initial-directory cases before mutating `context.includeDirectories`. Persistence flows through workspace settings so removals survive across sessions.
- `packages/cli/src/ui/commands/directoryCommand.test.tsx:+156` — 5 new tests pin the contract: validation rejects paths not in the include list, initial-directory guard fires, not-found returns a user-readable error, successful removal both updates state and persists settings, and settings-write failure is surfaced to the user (not swallowed). Coverage is appropriate for a user-facing slash command.
- `packages/core/src/utils/workspaceContext.ts:550` — `getSkippedDirectories(): readonly string[]` returns the internal tracker. The `readonly` return type is the right defensive choice; callers can't mutate the underlying array.
- `packages/core/src/utils/workspaceContext.test.ts:+54` — 4 tests for the new tracker covering: empty-when-all-exist, populated-with-missing, mixed-existing-and-missing, and reset-on-removal. Uses `fs.realpathSync` + `fs.mkdtempSync` per the file's existing convention to avoid macOS `/private` symlink quirks.
- `packages/core/src/config/config.ts:+6` — the startup `stderr` warning. Writing directly to `process.stderr` (rather than going through the structured logger) is the right call here because this fires before the TUI takes over the terminal; logger output at startup tends to get swallowed.
- `packages/cli/src/i18n/locales/en.js:+11` — 6 new strings for the remove subcommand, matching the existing key style. No translations for other locales added in this PR — assumed acceptable per the repo's i18n flow (English is source-of-truth and other locales are populated separately).
- `packages/cli/src/config/config.ts:381` — single-line help-text rewording for `--add-dir`. Now mentions path resolution and skip behavior. Good UX bandage for the silent-skip concern even before users see the new stderr warning.
- `.gitignore` — adds `.prforge/`, `.prforge-run`, `.prforge-*`. This is PR-tooling noise leaking into the repo's `.gitignore`. Maintainers will likely want this removed before merge — author's local PR-prep artifacts shouldn't be in the upstream `.gitignore`.

## Concerns

- **Unrelated changes bundled in.** The `relevanceSelector.ts` and `relevanceSelector.test.ts` changes (passing `config.getFastModel()` into `runSideQuery` for the auto-memory recall path) have nothing to do with the `--add-dir` polish described in the PR title. This duplicates qwen-code #3848 ("fix(memory): route auto-memory recall selector to fast model"), which is also currently open. Two PRs touching the same lines is a recipe for merge conflicts — the author should drop these changes from #3856 and let #3848 land them, or close #3848 and re-scope #3856's title/description.
- **`.gitignore` PR-tooling leakage.** As above, `.prforge*` patterns belong in the author's global gitignore, not the project's.
- **Startup warning emission timing.** Writing to `process.stderr` from `Config` construction is fine, but if `Config` is ever instantiated more than once during a session (e.g. by a re-config flow), users would see duplicate warnings. Worth a one-shot guard or just confirming `Config` is single-instance.

## Verdict

**request-changes**

## Rationale

The core feature work (remove subcommand, skipped-directories tracker, startup warning, help text) is solid, well-tested, and a clear UX improvement. But this PR conflates that work with an unrelated `relevanceSelector` change that overlaps a separate open PR (#3848), plus PR-tooling pollution in `.gitignore`. The right move is to split: keep #3856 focused on `--add-dir` polish per its title, drop the relevance-selector hunks (let #3848 own them), and remove the `.prforge*` `.gitignore` lines. After that split, the directory-command portion is cleanly mergeable.
