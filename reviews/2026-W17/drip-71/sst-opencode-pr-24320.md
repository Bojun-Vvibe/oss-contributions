# sst/opencode PR #24320 — match project-relative permissions in Read tool

- **PR**: https://github.com/sst/opencode/pull/24320
- **Author**: @pascalandr (Pascal André)
- **Head SHA**: `79a87ec83f1e72dda21c1ea76387bf1f07eeece4`

## Summary

Two-line behavior change in `packages/opencode/src/tool/read.ts:166-183`:
the permission `patterns` array passed to `ctx.ask({permission: "read"})`
now uses the project-relative `title` path when the file lives inside the
worktree, falling back to the absolute `filepath` only when the relative
path escapes the worktree (`startsWith("..")`) or is itself absolute on
the host:

```ts
const title = path.relative(Instance.worktree, filepath)
const permissionPath = title.startsWith("..") || path.isAbsolute(title)
  ? filepath
  : title
...
yield* ctx.ask({ permission: "read", patterns: [permissionPath], ... })
```

A 40-line regression test
(`packages/opencode/test/tool/read.test.ts:124-167`) constructs a
permission ruleset that denies an exact project-relative pattern
(`src/main/java/com/example/app/controller/ExampleController.java`),
reads the corresponding absolute path, and asserts the deny rule fires
through `Permission.evaluate`. A second adjustment fixes the existing
Windows test by mapping `C:` → `/C` and asserts `patterns` is now
the bare `test.txt` instead of the previously-absolute `full(target)`.

## Verdict: `merge-as-is`

This is a real correctness bug: until now, users writing
`config.permission.read = { "src/.../File.java": "deny" }` got nothing
because the matcher was always handed the absolute path
(`/abs/path/to/worktree/src/.../File.java`), which only ever matched
`/**/File.java`-style globs. Project-relative deny rules silently
no-op'd.

## Specific references

- `read.ts:167` — the `permissionPath` derivation. Sound: it preserves
  the existing absolute-path contract for files outside the worktree
  (`title` would be `../../something` or absolute on Windows when the
  drives differ) so external_directory permission flows are unchanged.
- `read.test.ts:124-167` — the new positive test. Constructs the
  ruleset via `Permission.fromConfig(...)` then checks that
  `Permission.evaluate(req.permission, pattern, ruleset)` returns
  `deny` for at least one pattern in `req.patterns`. Good shape.
- `read.test.ts:191-200` — the Windows path normalization tweak.
  `target.replace(/^([A-Za-z]):/, "/$1")` turns `C:\foo\test.txt`
  into `/C/foo/test.txt`. Then the assertion is `["test.txt"]` —
  i.e. project-relative on Windows too. Verify this matches what the
  tool actually emits when `Instance.worktree = "C:\foo"` and
  `filepath = "C:\foo\test.txt"`; the path.relative output is
  platform-dependent.

## Risks

1. **Symmetry with Write/Edit/Bash tools.** This PR fixes Read only.
   The same bug almost certainly affects `write.ts`, `edit.ts`, and
   any other tool that calls `ctx.ask({permission: ...})`. Ship this
   fix but file a follow-up to audit and unify.
2. **Pattern interpretation ambiguity.** `Permission.evaluate` must
   now handle both relative-segment patterns and absolute-path
   patterns interchangeably. If `evaluate` does substring or glob
   matching, that's fine; if it does exact-string matching, a
   pre-existing absolute-path config will stop matching after this
   change. Worth a one-line note in CHANGELOG / migration guide.

## Nits

- The `permissionPath` ternary is duplicated logic for "is this
  inside the worktree". A helper `pathRelativeToWorktree(filepath)`
  returning `string | null` (null when outside) would centralize
  this and make the follow-up to other tools mechanical.
