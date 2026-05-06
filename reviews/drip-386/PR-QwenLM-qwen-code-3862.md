# QwenLM/qwen-code#3862 — feat(skills): support nested skill directories

- **Head SHA**: `d0626c70f4b64f1d5c3838aead0a37b8e94de5c7`
- **Stats**: +395 / -104, 4 files

## Summary

Today's `loadSkillsFromDir` only enumerates the *top* level of a skills directory: a layout like `.qwen/skills/category/skill-name/SKILL.md` is invisible because the scanner does one `fs.readdir` and then looks for `SKILL.md` directly under each entry. This PR replaces the single-level scan with a depth-first recursive `collectSkillDirs` walk that yields every directory under the skills root, then probes each for a `SKILL.md`. Symlink-scope validation (existing security guard preventing skills from escaping the root) is preserved at every recursion step.

## Specific citations

- `packages/core/src/skills/skill-load.ts:24-110`: rewrite of the main loader. The pre-diff loop iterated `entries` once and called `validateSymlinkScope` per top-level entry; the new shape resolves `baseRealPath` once at line 28-39, then delegates the directory walk to a new `collectSkillDirs(baseDir, baseRealPath)` helper at line 90-130. The flat skill-load loop at lines 60-82 then probes each returned dir for `SKILL.md` via `fs.access`.
- `packages/core/src/skills/skill-load.ts:90-130`: new recursive helper. Two correctness points worth pinning: (1) **symlink-scope validation is propagated through every recursion frame** — at line 113 each symlink-typed entry is rechecked against the *original* `baseRealPath` (not against `dir`), so a deeply-nested symlink that points outside the original skills root is still caught. This is the load-bearing security property; SKILL.md files can ship hooks that execute shell, so a symlink-escape would be a code-execution vector. (2) The recursion fans out unconditionally on every directory and symlink-to-directory, with no depth limit and no cycle detection. A skills directory containing a symlink to itself (or a `.git`/`.snapshots`/`node_modules` subtree) would loop or do massive unnecessary IO. The third test case at `skill-load.test.ts:78-110` exercises 3 levels but doesn't exercise a self-pointing symlink; consider adding a depth bound (e.g. 8) or a visited-set keyed on `realpath`.
- `packages/core/src/skills/skill-load.ts:67-78`: error-handling tweak — `ENOENT`-or-`access`-string-match errors are now silently dropped (because every collected directory is probed for `SKILL.md`, and most directories *don't* contain one — that's expected, not an error). Other error categories still log via `debugLogger.error`. The string match `error.message.includes('ENOENT') || error.message.includes('access')` is brittle; a typed check (`error.code === 'ENOENT'` on `NodeJS.ErrnoException`) would be more durable across Node versions.
- `packages/core/src/skills/skill-load.test.ts:744-924` (new tests): four cases — basic 2-level nesting, 3-level deep nesting, mixed flat-and-nested layout, and the symlink-escape case at line 877+ where `fs.realpath` returns `/etc/escape-target` for the offending entry. The escape test asserts `skills.toHaveLength(0)`, pinning the security gate. The mocking pattern uses `mockResolvedValueOnce` chains keyed by recursion order, which is fragile to refactors of `collectSkillDirs`'s traversal order — a different walk order would silently break these tests; a more robust approach would use `mockImplementation` keyed by path argument.
- `packages/core/src/skills/skill-manager.test.ts:1531-1620`: parallel test additions on the `SkillManager` integration boundary. Test at line 1620 pins the symlink-escape case at the manager level too (defense-in-depth — `loadSkillsFromDir` and `SkillManager` both call `validateSymlinkScope`). Good.
- `packages/core/src/skills/skill-manager.ts:144-150`: tangential prettier-only reformatting of an `if` predicate. Not a code change. Worth squashing into "chore: prettier" rather than included in a feature PR, but harmless.

## Verdict

**merge-after-nits**

## Rationale

Real feature, real regression test, security gate (symlink scope) is preserved through recursion. Two nits worth addressing: (1) **add a depth limit or a visited-realpath set** to `collectSkillDirs` — the current shape walks unbounded and would fall into infinite recursion on a self-symlink loop or do gratuitous IO if a user accidentally symlinked their `~/skills` into their `node_modules` tree; an 8-level depth cap or a `Set<string>` of canonical-path-already-visited would make this robust. (2) Replace the substring-based `error.message.includes('ENOENT')` filter at lines 70-72 with a typed `(error as NodeJS.ErrnoException).code === 'ENOENT'` check — i18n'd error messages or future Node versions could break the substring match silently. (3) The unrelated prettier reflow at `skill-manager.ts:144` should split into its own commit. None block merge given the security property is preserved and the test coverage exercises it explicitly.
