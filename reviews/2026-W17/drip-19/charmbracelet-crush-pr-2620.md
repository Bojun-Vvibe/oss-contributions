---
pr_number: 2620
repo: charmbracelet/crush
head_sha: 7e6c14e92534440f2dcba9b4098cc60f8ffaa0da
verdict: merge-after-nits
date: 2026-04-25
---

# charmbracelet/crush#2620 — `crush skills list` command with group-by-source

**What changed.** 4 new files / +614 / −0. `internal/cmd/skills.go` (210 lines) defines `skillsCmd` (with `skill` alias) and `skillsListCmd`, supporting `--flat`, `--json`, and positional-arg search filters. Tree output uses `lipgloss/v2/tree`, headers in `charmtone.Malibu`, disabled markers in `charmtone.Damson`. `internal/skills/group.go` exports `GroupBySource` and `ClassifySource`. Tests in `skills_test.go` (+ `group_test.go`, 6 cases) cover classification and output formatting; non-TTY auto-falls-back to flat.

**Why it matters.** Until now there was no way to introspect which skills the runtime had discovered or where they came from. With `~/.config` skills, project-local skills, and built-ins all merging through `Deduplicate`, the only way to debug "why isn't this skill showing up" was to read source.

**Concerns.**
1. **`outputSkillsTree` calls `groupNode := tree.Root(headerStyle.Render(string(g.Source)))` then `t.Child(groupNode)`** (skills.go line 174ish). `tree.New()` returns the trunk; making each group its own `tree.Root` and then nesting is unusual — most lipgloss tree examples use `tree.Root("header").Child(...)` for the top-level. Verify that the rendered output actually looks like the example in the PR body (`├── Builtin / └── Project`) and not a degenerate single-level tree. The example screenshots help but not a substitute for running locally on a multi-group setup.
2. **`disabledStyle.Render("(disabled)")` is appended after the description string.** That means a skill with no description renders as `name  (disabled)` (double space, no em-dash). Cosmetic but visible.
3. **`if term != "" { ... }` filter (line ~73)** runs `strings.ToLower` on every skill name and description per call. Fine at current skill counts but if `Discover` ever returns thousands (project-tree walks?), pre-lowercasing once at decode time would help. Not blocking.
4. **`return fmt.Errorf("no skills found matching %q", term)`** for an empty filter result returns a non-zero exit. For a `list` command this is hostile to scripts — `crush skills list nonexistent | head` will fail the pipeline. Consider exit-0 with empty output (matching `git branch --list nonexistent`).
5. **JSON output emits `path` field per `skillJSON`** but the diff truncation cut before I could verify `path` is populated for all sources. If `ClassifySource` returns `Builtin` for an in-binary skill, `Path` should probably be empty string rather than `<nil>`. Test does cover this — recommend asserting `Path != ""` for non-builtin.
6. **No completion / man-page / docs touch.** `crush models list` likely had docs added when introduced; mirror that for parity.

Useful operator surface, clean test coverage, follows existing patterns. Land after the exit-code nit (#4) is addressed.
