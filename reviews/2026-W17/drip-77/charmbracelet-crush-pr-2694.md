---
pr: 2694
repo: charmbracelet/crush
sha: 0491832a3c54
verdict: merge-as-is
date: 2026-04-26
---

# charmbracelet/crush #2694 — fix(skills): deduplicate skills discovered via symlinked directories

- **URL**: https://github.com/charmbracelet/crush/pull/2694
- **Author**: octo-patch
- **Head SHA**: 0491832a3c54
- **Size**: +49/-2 across 2 files (`internal/skills/skills.go`, `internal/skills/skills_test.go`)

## Scope

Fixes duplicate skill loading when the same on-disk skill file is reachable via two discovery paths that resolve to the same target through a symlink — e.g. `~/.config/crush/skills` symlinked to a project's `.claude/skills`, with both paths registered as discovery roots.

The fix at `skills.go:213-228`:
- After `filepath.WalkDir` reaches a `SKILL.md`, run `filepath.EvalSymlinks(path)` and use the resolved path as the dedup key (`realPath`) for the `seen` map.
- Fall back to `path` if `EvalSymlinks` errors (preserves existing behavior for the no-symlinks-involved case).
- Continue to call `Parse(path)` (the original path), so error messages and discovered metadata still reference the discovery-relative path the user wrote.

Test at `skills_test.go:285-326`:
- Builds a real skills dir + a symlink dir pointing at the same target.
- Calls `Discover([realDir, symlinkDir])`.
- Subscribes to `SubscribeEvents` and asserts `len(skills) == 1` and exactly one `StateNormal` state event was emitted.
- Comment notes "Not parallel: shares global broker with other Discover tests".

## Specific findings

- **Right semantics: dedup by resolved target, parse by discovery path.** This preserves the user's mental model ("the skill I see in my list came from path X") while preventing the broker from double-counting. Excellent split.

- **Fall-through when `EvalSymlinks` errors is conservative-correct.** A broken symlink, permission error, or `EvalSymlinks` returning a path that doesn't exist would all fall back to the original `path` as the dedup key — same as pre-fix behavior. So the worst-case post-PR behavior is "no improvement" rather than "skipped skill".

- **Locking shape is preserved.** `mu.Lock` / check-and-set-`seen` / `mu.Unlock` happens before `Parse`, which is fine because `Parse` is the slow part and we don't want to hold the mutex across IO. The original code had this shape; the fix only changes which key is checked. No new race.

- **`filepath.EvalSymlinks` cost concern is minor.** It's one `lstat`+`readlink` per discovered `SKILL.md` (not per file walked). For repos with O(10) skills, this is negligible. For a hypothetical user with hundreds of skill dirs, it's still bounded by the count of `SKILL.md` files, not directory entries.

- **Test gates correctly: `Not parallel`.** Skills tests share a global broker, and parallel tests would race on the event subscription. The author noted this matches the convention in the rest of the file. Good.

- **Test cleanup**: `t.TempDir()` handles teardown, including the symlink (Go's `os.Symlink` creates a symlink, and `t.TempDir`'s cleanup `os.RemoveAll` removes symlinks without following them). Correct.

- **Edge case not covered**: what if `realDir` is itself reachable via two different symlinked routes from outside the workspace? E.g. `/Volumes/work` symlinks to `/Users/x/work` on macOS and discovery includes both — `EvalSymlinks` resolves to the same canonical path, dedup works. ✓
- **Edge case partially covered**: what if two *different* skill files have the same name (`SKILL.md` is the only filename matched per `:215`) but live in different real dirs that happen to alias? The dedup key is the *resolved* path of the file itself, so two truly distinct files have two different `realPath` values and both load. ✓

- **Path-canonicalization choice.** `filepath.EvalSymlinks` resolves all components, but doesn't apply OS-specific case-folding (mac/HFS+/APFS) or unicode normalization. On case-insensitive filesystems, two paths differing only in case would resolve to two different `realPath` values and both load — minor, almost certainly out of scope, but worth a one-line comment if anyone has reported it.

- **`Parse(path)` still called with the un-resolved path.** Confirms the user-visible "where this skill came from" stays user-friendly (their configured discovery dir, not the resolved symlink target). If `Parse` writes errors with the path, those errors stay in the user's mental model. Good UX.

- **Comment quality at `:213-217`** is exactly right: identifies the user-visible scenario (`~/.config/crush/skills` symlinked to project's `.claude/skills`), explains the fix in one sentence, points at the symptom. Future maintainer will read this and understand.

- **No drive-by changes**, no formatting churn, no whitespace edits. Tightly scoped diff.

## Risk

Trivially low. Pure addition of a defensive symlink resolution + a regression test. Failure mode is "user with a symlink loop has skills listed once instead of twice", which is unambiguously the intended behavior. No callsite of `Discover` cares about duplicates being silently dropped (the deduplication already existed; this just makes the dedup key correct under symlinks).

## Nits

1. (Optional) one-line comment noting case-insensitive-filesystem edge case is not handled.
2. (Optional) consider documenting the symlink-resolution policy at the package level (`internal/skills/doc.go` or similar) so future contributors know "we dedup on the resolved target, not the discovery path".

## Verdict

**merge-as-is** — minimal, correctly-shaped, covered by a focused regression test, with a comment that explains the user scenario. Closes a real footgun for any user whose config directory is a symlink to a project skills directory.

## What I learned

`filepath.EvalSymlinks` is the right primitive for filesystem-deduplication. The pattern of "resolve for dedup, original for display" — `realPath` for the `seen` map but `path` for `Parse` — is the same one good archive extractors use ("validate against absolute path, present relative path to user") and Linux mount-namespace tools use ("kernel sees the resolved path, user sees the bind-mount path"). Whenever you have a path used for two purposes (uniqueness + identity), using two different forms of it is almost always correct.
