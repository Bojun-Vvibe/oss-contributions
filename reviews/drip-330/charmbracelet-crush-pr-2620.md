# charmbracelet/crush #2620 — feat(cmd): add `crush skills list` command with group-by-source support

- SHA: `7e6c14e92534440f2dcba9b4098cc60f8ffaa0da`
- State: OPEN, +614/-0 across 4 files
- Related: charmbracelet/crush#2608

## Summary

Adds a new `crush skills` subcommand tree with one initial leaf (`list`) that discovers all skills, deduplicates, and renders them grouped by source (Builtin / Project / User) using the existing `lipgloss` tree pattern. Supports `--flat`, `--json`, positional name/description filtering, and auto-falls-back to flat for non-TTY pipes. Ships +340 LOC of tests across 28 cases.

## Notes

- `internal/cmd/skills.go:74-118` (`runSkillsList`) — output mode selection chain is `JSON > flat > non-TTY > tree`. The non-TTY branch (`!isatty.IsTerminal(os.Stdout.Fd())`) at line 113 falls through to `outputSkillsFlat`, which is the right default for `crush skills list | grep`. One subtle gap: `--flat --json` together silently picks JSON because line 110 checks JSON first — fine, but worth either documenting precedence in the long help or adding a `cobra.MutuallyExclusiveFlags("flat", "json")` registration in `init()`.
- `internal/cmd/skills.go:121-141` (`discoverAllSkills`) — runs `skills.DiscoverBuiltin()`, then optionally appends `skills.Discover(opts.SkillsPaths)`, then `skills.Deduplicate(...)`. Dedup happens after merge — correct. `home.Long(pth)` expansion is applied per-path at line 130; verify this matches the path semantics used by the actual loader at runtime, otherwise the listed paths will diverge from what the agent actually loads.
- `internal/cmd/skills.go:166-189` (`outputSkillsTree`) — uses `charmtone.Malibu` for headers and `charmtone.Damson` for the `(disabled)` badge. PR body claims this matches "session list styling conventions". Worth grepping `internal/cmd/session*.go` to confirm; if the session list uses different tones, harmonize before merging so the TUI palette stays coherent.
- `internal/cmd/skills.go:192-205` (`outputSkillsFlat`) — uses `tabwriter` with min-width 0, padding 2. Three columns: name, source (lowercased), status. No header row. Adding a header row (or a `--no-header` flag) helps shell users who pipe to `awk` or `cut -f1`. Optional.
- `internal/cmd/skills.go:218` — JSON output struct `skillJSON` includes `Path`. The PR's example shows `"path":"crush://skills/crush-config/SKILL.md"` for builtin skills — confirm this URL scheme is stable; if it ever changes (e.g. to a real filesystem path for builtin embeds), JSON consumers break.
- `internal/skills/group.go` (new, +64) — `GroupBySource` and `ClassifySource` are the load-bearing helpers. Not visible in the abbreviated diff above, but the corresponding `group_test.go` (+133, 6 cases per PR body) should cover at minimum: builtin classification, project classification (path matches `projectDirs`), user fallback, and the precedence between builtin embedded path and any user override of the same name.
- `internal/cmd/skills_test.go` (+207, 5 cases) — covers filter and output formatting per PR body. Suggest adding one integration-style test that runs `runSkillsList` end-to-end with a tempdir-scoped project skill to catch regressions in the `ProjectSkillsDir(cwd)` plumbing on line 89.
- Error UX: line 99 returns `fmt.Errorf("no skills found matching %q", term)` for empty filtered result. This makes the command exit non-zero on a legitimate "no match" outcome. Most CLIs treat empty search results as exit 0 with an empty stdout (or a stderr message). Worth confirming this matches the rest of `crush`'s convention — `crush models` likely doesn't error on empty filters.
- Nit: line 39 `Args: cobra.ArbitraryArgs` plus line 86 `term := strings.ToLower(strings.Join(args, " "))` joins multi-arg search with space. `crush skills list foo bar` therefore searches for the literal substring `foo bar`, which is probably surprising — most users would expect AND-of-tokens. Either document this in `--help` or split-and-AND.

## Verdict

`merge-after-nits` — clean, well-tested, follows existing `crush models` patterns. Land after: (1) document `--flat` vs `--json` precedence (or mark mutually exclusive), (2) confirm `charmtone.Malibu/Damson` matches actual session-list styling, (3) decide whether "no matches" should be exit-0 or exit-1 to match repo convention, (4) clarify multi-token positional search semantics in help text.
