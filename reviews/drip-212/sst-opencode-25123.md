# sst/opencode PR #25123 — fix: ensure disabling OPENCODE_DISABLE_CLAUDE_CODE_SKILLS doesnt disable external skills too

- Repo: `sst/opencode`
- PR: https://github.com/sst/opencode/pull/25123
- Head SHA: `09b005a9784b5cad61e11fe0911f8a045a414c29`
- State: MERGED (2026-04-30T16:15:53Z)
- Files: 2 (`packages/core/src/flag/flag.ts`, `packages/opencode/src/skill/index.ts`), +6/−3
- Verdict: **merge-as-is** (already merged; this is a post-merge note)

## What it does

Untangles two skill-disable env vars whose composition was wrong. Before:

```ts
OPENCODE_DISABLE_EXTERNAL_SKILLS:
  OPENCODE_DISABLE_CLAUDE_CODE_SKILLS || truthy("OPENCODE_DISABLE_EXTERNAL_SKILLS")
```

i.e. setting `OPENCODE_DISABLE_CLAUDE_CODE_SKILLS=1` (intent: hide
`~/.claude/skills/**`) silently set `OPENCODE_DISABLE_EXTERNAL_SKILLS=1`,
which also dropped `~/.agents/skills/**` from discovery — the user-visible
"disable claude-code skills" toggle nuked the unrelated `.agents` tree.

The fix decomposes `EXTERNAL_DIRS = [".claude", ".agents"]` (the prior
single bag in `packages/opencode/src/skill/index.ts:23`) into two named
constants and gates `.claude` independently of `.agents`:

```ts
const externalDirs: string[] = []
if (!Flag.OPENCODE_DISABLE_EXTERNAL_SKILLS) {
  if (!Flag.OPENCODE_DISABLE_CLAUDE_CODE_SKILLS) externalDirs.push(CLAUDE_EXTERNAL_DIR)
  externalDirs.push(AGENTS_EXTERNAL_DIR)
  // ...used for both `Global.Path.home` scan and `fsys.up({ targets: externalDirs })`
}
```

Paired with the `flag.ts` change that breaks the `OPENCODE_DISABLE_CLAUDE_CODE_SKILLS ||`
short-circuit, so the two flags are now orthogonal.

## Notes

- The old `EXTERNAL_DIRS` bag was used in two places (`scan` of
  `Global.Path.home` and `fsys.up({ targets: EXTERNAL_DIRS, ... })`); the
  fix routes both through the locally-built `externalDirs` array, so
  there's no second site that still iterates `[".claude", ".agents"]`
  unconditionally.
- Right shape for a "two flags compose as AND-of-not, not OR-of-not"
  bug: gate at the dir-list construction site, not at the loop body, so
  every downstream consumer (the home-scan loop and the worktree-up
  walker) automatically inherits the new policy.
- The flag matrix is now {EXT_OFF, CC_OFF} → 4 cells: (0,0)=both shown,
  (0,1)=only `.agents`, (1,0)=neither (because `EXTERNAL` gates the
  whole block), (1,1)=neither. That matches what users would naively
  expect from the env-var names.

## Nits (post-merge follow-ups)

1. No regression test added. A 4-row table-driven test over
   `(OPENCODE_DISABLE_EXTERNAL_SKILLS, OPENCODE_DISABLE_CLAUDE_CODE_SKILLS)`
   asserting the resulting `externalDirs` would have caught the
   original `||` short-circuit and would prevent re-regression.
2. Subtle ordering: `.claude` is pushed before `.agents`, which means
   if both have a `SKILL.md` of the same slug, `.claude` wins by
   first-occurrence-wins in the dedup pass. Worth a one-line comment
   stating the precedence is intentional (or confirming it's not
   load-bearing).
3. The `OPENCODE_DISABLE_CLAUDE_CODE_PROMPT: OPENCODE_DISABLE_CLAUDE_CODE || truthy(...)`
   composition at `flag.ts:48` is structurally identical to the bug
   that was just fixed for `_SKILLS`. Worth grepping the rest of the
   file for the same `parent || truthy(child)` pattern and confirming
   each instance is intended (the `_PROMPT` one probably is, since
   "disable claude code entirely" should imply "disable its prompt
   surface", but it warrants a comment).

## Why merge-as-is

Already merged. Two-line semantic fix to a real user-visible bug, the
shape (gate at construction site, not loop body) is right, and the
diff is minimum-blast-radius.
