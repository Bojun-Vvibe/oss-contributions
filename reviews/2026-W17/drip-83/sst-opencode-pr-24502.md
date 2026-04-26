# sst/opencode PR #24502 — fix: add logging to silent catch block in workspace restore bootstrap

- Repo: sst/opencode
- PR: #24502
- Head SHA: `8dfc9a79`
- Author: @alfredocristofano
- Diff: targeted +2/-1 on `packages/opencode/src/cli/cmd/tui/component/dialog-workspace-create.tsx` (PR also drags an unrelated `AGENTS.md` rewrite + Config import-path migration across ~30 files into the diff — same noise pattern as #24503/#24504 from this contributor)
- Closes: #24327

## What changed

The substantive change is at `dialog-workspace-create.tsx:143-149`: the previously empty `} catch (e) {}` around `await input.sync.bootstrap({ fatal: false })` becomes `} catch (e) { log.warn("workspace bootstrap failed", { error: errorData(e) }) }`. Restores diagnosability of bootstrap failures during workspace session restore — the call passes `fatal: false` precisely because the caller wants to continue, but the silent swallow made the failure path invisible.

## Specific observations

- The PR description correctly notes that `errorData` and `log` are already imported in the file (used by line 149 in the surrounding context), so this is pure use of pre-existing utilities — no new dep, no shape change to the catch contract.
- `log.warn` is the right level: the call site uses `fatal: false` so the failure is recoverable, but `log.debug` would be too quiet for "workspace failed to bootstrap on restore." Matches the convention used in the sibling `try { await input.sync.subscribe(...) } catch (e) { log.warn(...) }` block already in this file.
- Same diff hygiene problem as the other Alfredo PRs in this round (#24503, #24504): the actual targeted change is one hunk, but the PR carries ~98 unrelated files of `Config` import migration + `AGENTS.md` rewrite + `infra/app.ts` regen. Maintainers should ask the contributor to land the import-migration + AGENTS.md as separate PRs so the per-file `catch{}` cleanups can be cherry-picked independently. The MCP catch fix (#24503) already documents this pattern.

## Verdict

`merge-after-nits`

## Rationale

The substantive change is correct and trivially safe. The blocker is diff hygiene — 98 unrelated files riding along makes review of the workspace-restore behavior harder than it needs to be. Ask for a clean rebase that strips the Config import migration into its own PR.
