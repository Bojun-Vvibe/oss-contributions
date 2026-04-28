# Review — sst/opencode #24728

- **Title**: feat: `opencode session move` / `session detached`
- **Author**: rektide
- **Head**: `f343c95a033a36423dfeab5a956238f72fa72467`
- **Verdict**: needs-discussion

## Summary

Adds two new CLI subcommands (`session move`, `session detached`) at `packages/opencode/src/cli/cmd/session.ts:55-275` to migrate sessions across projects/directories and surface sessions whose stored `directory` no longer matches the on-disk reality. Closes #24708. Lands the same week as the much larger #24726 (`session migrate`/`orphans`/`rebind` with HTTP API + SDK + global session listing) — these two PRs are the *competing* design responses to the orphan-recovery problem from #23250.

## Findings

- **Direct-SQL bypass of the Session service layer** at `:144-149` (`Database.use((db) => db.update(SessionTable).set(set).where(where).run())`) sidesteps every invariant `Session.update()` enforces (cache invalidation, observers, `updated_at` touch, side-effects on child sessions). #24726 takes the opposite approach via `Session.migrate()` going through the service. If both ship, the direct-SQL path will silently regress when service-layer guarantees evolve. Either route through `Session.Service.use((svc) => svc.move(...))` or document why the bypass is intentional.
- **Child-session tree is not migrated** — `move` mutates only the matching session rows; sessions with `parent_id` pointing at a moved row keep their old `directory`/`project_id`. #24726's `Session.migrate()` explicitly walks the child tree. For multi-turn agent sessions with subagents, this leaves an inconsistent forest after a `move`. Add a child-walk or document the limitation prominently in `--help`.
- **Glob-to-LIKE translation at `:118` is unsafe** — `dir.replace(/\*/g, "%").replace(/\?/g, "_")` does not escape literal `%`/`_` already present in the path (legitimate on POSIX), so `--from-dir "/home/user/100%proj"` matches unintended directories. Escape `%` and `_` in the source string before substituting wildcards.
- **No `--to-project` validation** — `:140` accepts any string and writes it to `project_id`; a typo silently creates a session pointing at a nonexistent project. Validate via `Project.get()` or document that `move` is intentionally lenient.
- **`detached` `Promise.all` over all sessions at `:248-253`** with a per-directory `git rev-parse`/`git rev-list` shell-out — for users with thousands of sessions across hundreds of dirs the directory cache helps but there's no concurrency cap; on macOS this hits the per-process file descriptor limit. Cap with `p-limit` or batched chunks.
- **Resolution semantics divergence from #24726**: `resolveDir` at `:182-208` derives `projectID` from `git rev-list --max-parents=0 HEAD | sort | head -1` (first root commit lex-min), but #24726 detects orphans by comparing against `Instance` resolution. Two different "what project does this directory belong to" oracles in the same release will drift. Reconcile or alias one to the other.

## Recommendation

Hold pending coordination with #24726 — the two PRs solve overlapping problems with incompatible primitives (CLI-only direct-SQL vs. service-layer + HTTP API + SDK). Maintainers should pick one storage path; both can plausibly keep their CLI surface but they must share the underlying mutator. Independently of that, the glob-escape and child-tree gaps are real correctness issues that need fixing in either world.