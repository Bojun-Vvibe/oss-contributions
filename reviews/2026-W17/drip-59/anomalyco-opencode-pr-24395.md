# anomalyco/opencode #24395 — feat(memory): add agent_memory table and memory-tools plugin

- URL: https://github.com/anomalyco/opencode/pull/24395
- Head SHA: `ec84337eb5913a2b1cba68e73208821ec88e5fbb`
- State: OPEN
- Size: +3108/-49 across 18 files
- Verdict: **needs-discussion**

## What it does

Two coupled landings:

1. **Core**: a new `agent_memory` SQLite table + `AgentMemory` service
   in `packages/opencode/src/session/memory.ts:1-101` with `save` /
   `search` / `consolidate` operations, plus an `AgentMemoryID` brand
   in `packages/opencode/src/id/id.ts` and a Drizzle migration at
   `packages/opencode/migration/20260425194233_agent_memory/`.
2. **Plugin**: a brand-new `packages/memory-tools` workspace package
   that pushes/pulls memories to a Supabase project for cloud
   backup/restore, with a Supabase migration in
   `packages/memory-tools/supabase/migrations/001_agent_memory.sql`.

## Diff notes

Core service (`session/memory.ts:24-50`) does what it says — insert
on `save` with auto-generated descending ID, status defaults to
`"active"`, `time_created`/`time_updated` set to `Date.now()`. Search
takes `{type, keyword, limit}` and uses Drizzle `like` over a `%kw%`
pattern joined with `AND`. `consolidate` flips rows older than a
cutoff (default 30 days) from `active` → `consolidated`.

The Supabase migration in
`packages/memory-tools/supabase/migrations/001_agent_memory.sql`
references "FTS5 full-text search" in the README, but the actual
on-disk SQLite migration in
`packages/opencode/migration/20260425194233_agent_memory/migration.sql`
is a plain table with B-tree indexes — the search path uses `LIKE`,
not FTS5. The README claim is therefore wrong for the local store.

## Concerns (why needs-discussion, not merge-after-nits)

1. **Plugin package landing in monorepo without consumer wiring.**
   `packages/memory-tools` is a full new workspace (READMEs, ts
   configs, Supabase migration, sync.ts) but I see no entry in the
   root manifest or default plugin list. It'll only run if a user
   opts in via `opencode.json`. That's fine *if intentional*, but the
   PR mixes "core schema change" (committed migration that *every*
   user will pick up on next launch) with "optional cloud plugin"
   (only run by users who configure Supabase). These should land
   separately — the core table is harmless and reviewable; the
   plugin is a 1500-line cloud-sync surface that deserves its own
   PR with its own threat model.

2. **Privacy / egress.** AgentMemory captures `what/why/where/learned`
   metadata that may contain code paths, secrets, or proprietary
   context. Pushing that to a user-configured Supabase is a real
   data-egress channel and the PR body / README don't discuss:
   - opt-in default (it looks opt-in via plugin install — good)
   - what's filtered before upload (nothing visible in `sync.ts`
     skim — entire row including `metadata`/`tags`/`content` ships)
   - encryption-at-rest in Supabase or just "user trusts their RLS
     policy"
   At minimum this needs a clear warning in the README and a
   redaction hook before the row leaves the local DB.

3. **`db<T>` helper at `memory.ts:23` is mis-typed.** The
   `Parameters<typeof Database.use>[0] extends (trx: infer D) =>
   any ? D : never` conditional is doing manual type extraction in
   userland; this kind of helper belongs in `@/storage` so every
   future caller doesn't reinvent it. Not blocking but flagged.

4. **`consolidate` semantics are silently destructive.** It flips
   *all* `active` rows older than 30 days to `consolidated` with no
   guardrail. There's no opposite operation in this PR, no UI for
   reviewing what's about to be consolidated, and the default cutoff
   is hardcoded. For a memory system, "30 days = automatically
   demoted" is a strong opinion that deserves config.

5. **Test coverage gap.** I see fake-provider plumbing
   (`test/fake/provider.ts`, `test/session/fallback.test.ts` updates)
   but no direct test of the new `AgentMemory.save/search/consolidate`
   round-trip, no migration-rollback test, and no Supabase sync
   contract test (mocked).

## Why this verdict

The core idea is sound and well-aligned with existing OpenCode
patterns (Effect service, Drizzle table, descending-ID brand). But
the PR conflates a load-bearing schema migration with an optional
cloud-backup plugin, lacks privacy review of the egress surface, and
ships consolidation defaults without a way back. Split the PR, then
this can be reviewed properly.
