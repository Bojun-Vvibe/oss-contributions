# charmbracelet/crush PR #2694 — fix(skills): deduplicate skills discovered via symlinked directories

- URL: https://github.com/charmbracelet/crush/pull/2694
- Head SHA: `0491832a3c54d83f4126bc35fbdb90221b447e8e`
- Author: octo-patch
- Repo: charmbracelet/crush

## Summary

`DiscoverWithStates` keyed its de-dup `seen` map on the literal walked path,
so the same skill reachable via two roots (e.g. `~/.config/crush/skills` is a
symlink into a project's `.claude/skills`) loaded twice. PR resolves the
symlink before the membership check.

## Specific notes

- `internal/skills/skills.go:216-228` — `filepath.EvalSymlinks(path)` is
  called inside the walk callback; on error the code falls back to the
  original path. That is the right safety: a broken symlink shouldn't
  cause discovery to silently drop a skill. Good.
- Note: `EvalSymlinks` walks every symlink in the path, including parent
  components, which is what we want for the "two roots pointing to the
  same directory" case described in the test.
- Performance: per-file syscall added inside the walk. For typical skill
  trees (dozens of files) this is negligible; if discovery ever extends
  to large unrelated trees this would be the obvious thing to revisit.
- The actual `Parse(path)` call still uses the unresolved `path`
  (line 234, just below the diff). That's intentional — keeps log paths
  user-recognizable — and only `seen` membership uses `realPath`. Good.
- `skills_test.go:286-322` `TestDiscoverDeduplicatesSymlinks` constructs
  a real dir + a symlinked dir pointing to it, calls `Discover` with
  both roots, and asserts `len(skills) == 1` and exactly one
  `StateNormal` event payload. That covers the regression. The test is
  marked non-parallel with a comment about the global broker — matches
  the convention of the surrounding tests. Skipping on Windows is not
  needed because `os.Symlink` failures would already surface; could add
  `if runtime.GOOS == "windows" { t.Skip(...) }` defensively if Windows
  CI is flaky on symlink perms.

verdict: merge-as-is
