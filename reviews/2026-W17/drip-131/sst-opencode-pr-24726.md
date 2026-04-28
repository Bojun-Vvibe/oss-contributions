# Review — sst/opencode #24726

- **Title**: feat(session): add methods to migrate session
- **Author**: Alchuang22-dev (Zeyu Zhang)
- **Head**: `53dba0482292f49710922573c05e8465942b0dde`
- **Verdict**: merge-after-nits

## Summary

Lands the backend half of the orphan-recovery design from #23250: adds `Session.listGlobal()`, `Session.listOrphans()`, `Session.migrate()` (going through the service layer), HTTP routes `GET /session/history`, `GET /session/orphans`, `POST /session/:sessionID/migrate`, JS SDK regeneration, and three CLI subcommands (`session list --all`, `session orphans`, `session migrate`/`rebind`) at `packages/opencode/src/cli/cmd/session.ts:49-235`. Explicitly defers the TUI migration dialog from #23250.

## Findings

- **Right architectural shape**: `migrate` goes through `Session.Service.use((svc) => svc.migrate(input))` at `:213-215` rather than direct SQL, preserving cache invalidation, observers, and child-tree handling. Contrast with #24728 which writes `SessionTable` directly. If both PRs land, this one's primitives should be the canonical mutator.
- **Project-resolution branch at `:191-204`**: when `--project` is omitted, derives `projectID` via `Instance.provide({directory, fn: ...})` which is the standard resolver — same code path as fresh-session creation, so a migrated session will be classified the same way as a session that was *originally* created in the target dir. Good consistency invariant.
- **`Row` type union at `:223`** (`Session.Info | Session.GlobalInfo`) and the `projectName(s)` narrowing at `:227-230` (`if (!("project" in session)) return ""`) is the right shape for the dual-mode table renderer; the conditional column-width math at `:235` correctly only allocates the Project column when `all=true` so non-`--all` tables stay byte-identical to today's output (no regression for existing scripts parsing the table).
- **Pagination cursor mentioned in PR body but not in CLI** — `--all` does not accept a `--cursor` flag, so users with very large session histories can't page beyond `--max-count`. The HTTP API has cursor support per the body; surface it on the CLI or document the limitation.
- **`SessionMigrateCommand` JSON output at `:218-220`** prints a single-element array `formatSessionJSON([session])` — most other JSON-output paths in opencode emit a single object for single-result endpoints. Consider matching the rest of the surface or document the wrapping for SDK consumers parsing `migrate` output.
- **No test for the orphan-detection edge case** where a global session's directory *becomes* a known-project worktree after the fact (the second orphan-class named in the PR body). The `bun test test/server/global-session-list.test.ts` cited in verification is a good start; add a focused fixture for the "global → reattachable" transition because that's the bug class users will actually file.
- **`SessionMigrateCommand` aliases includes `rebind`** at `:165` — good for discoverability but the success message at `:223-227` always says "migrated to" regardless of which alias the user typed. Minor: phrase as "migrated/rebound" or reflect the chosen alias.

## Recommendation

Merge after wiring `--cursor` for `session list --all` (or documenting the limitation), adding the global-session-becomes-project orphan test, and aligning the JSON output shape for `migrate`. The architectural decisions are right and this should be the canonical path for orphan recovery; coordinate with #24728 to settle on a single storage mutator before both land.