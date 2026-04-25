# charmbracelet/crush #2694 — fix(skills): deduplicate skills discovered via symlinked directories

- **Repo**: charmbracelet/crush
- **PR**: [#2694](https://github.com/charmbracelet/crush/pull/2694)
- **Head SHA**: `0491832a3c54d83f4126bc35fbdb90221b447e8e`
- **Author**: octo-patch
- **State**: OPEN (+49 / -2)
- **Verdict**: `merge-as-is`

## Context

Issue #2683: when `~/.config/crush/skills` is a symlink into a
project's `.claude/skills/`, `fastwalk` (run with `Follow: true`) walks
the same physical files twice — once under each logical root — and the
sidebar renders both `SkillState` entries because the `seen` map keys on
the raw walk path. The skill registry itself dedups by name, so
`crush_info` shows the right count, but the UI lists every skill twice.
Pure cosmetic / state-bookkeeping bug; no correctness or security
impact.

## Design

The fix at `internal/skills/skills.go:213-227` is exactly what you'd
write:

```go
realPath := path
if resolved, resolveErr := filepath.EvalSymlinks(path); resolveErr == nil {
    realPath = resolved
}
mu.Lock()
if seen[realPath] {
    mu.Unlock()
    return nil
}
seen[realPath] = true
mu.Unlock()
```

`EvalSymlinks` collapses any chain of symlinks down to a canonical
physical path, so two logical walks landing on the same `SKILL.md`
share a key. The fallback ("if `EvalSymlinks` fails, use the original
path") is the right call — it preserves the previous behavior on
dangling symlinks rather than silently dropping skills, and the previous
behavior was already to load the file (which would then fail at parse
time anyway, surfacing the broken symlink to the user).

`Parse(path)` is intentionally still called with the original `path`,
not `realPath`. That's correct — we want the user-facing skill metadata
(directory name, etc.) to reflect how the skill was discovered, not
where it physically lives. Good attention to detail.

## Test

`TestDiscoverDeduplicatesSymlinks` at `skills_test.go:287-326` builds a
real dir + symlink dir pointing at the same target, calls `Discover`
with both, and asserts `len(skills) == 1` plus exactly one
`StateNormal` event. Tight, deterministic, no flakes.

The "Not parallel: shares global broker with other Discover tests"
comment is honest about the constraint.

## Risks / Nits

1. **`EvalSymlinks` can be slow on Windows** for paths with many
   intermediate symlinks (it issues `GetFinalPathNameByHandle` calls).
   For a typical skills tree this is negligible (one stat per
   `SKILL.md`), but worth noting if anyone later widens this pattern to
   the entire fastwalk callback. Not a blocker here.

2. **Race with on-disk symlink swap** — between `EvalSymlinks(path)`
   and `Parse(path)`, the symlink could in principle change. Same race
   exists on `main`, so this PR doesn't make it worse. Not worth
   guarding.

3. Nit: the variable shadow with the outer `seen` (declared somewhere
   above the diff window) is fine, but a one-line comment above the
   `seen[realPath] = true` line clarifying that the map keys are now
   physical paths would help future readers. Optional.

## Verdict rationale

`merge-as-is`. Tight scope, correct fix, regression test that exercises
the exact reported topology, no behavior change for the non-symlink
case. The fallback-on-`EvalSymlinks`-error is the right conservative
choice. Ship it.

## What I learned

The `seen` map in a parallel walker that follows symlinks should
*always* be keyed by `EvalSymlinks` output, not the walk path. This is
a recurring footgun any time you opt into `fastwalk` / `filepath.Walk`
with symlink following — worth a lint rule if anyone wants to write
one.
