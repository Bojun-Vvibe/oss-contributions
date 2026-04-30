# QwenLM/qwen-code PR #3743 — fix(cli): prevent file paths from being treated as slash commands

- **URL:** https://github.com/QwenLM/qwen-code/pull/3743
- **Head SHA:** `776a2ab41cffb56e5ccaa5b1503e659074a1b683`
- **Diff size:** +163 / -3 across 4 files
- **Verdict:** `merge-after-nits`

## Specific diff references

- `packages/cli/src/ui/utils/commandUtils.ts:38-67` — new `looksLikeCommandName(name)` predicate. Implementation reads as a single regex over `[A-Za-z0-9_\-:.]+` (callable name characters) — colons for MCP-style `mcp:server__tool`, dots for extension-qualified `gcp.deploy`, hyphens/underscores/digits for normal names. Anything containing `/`, `\`, whitespace, `@`, `#`, or non-ASCII fails the check, which is the right denylist for the "is this a file path or a command?" disambiguation. The choice to allow `:` inside is what keeps `mcp:server__tool` working — important to call out because an over-eager rewrite could regress that case.
- `packages/cli/src/ui/hooks/slashCommandProcessor.ts:449-459` — the actual fix at the dispatch site. After the `isSlashCommand` gate, a second check runs only on `/`-prefixed input: it pulls the first whitespace-delimited token after the slash and runs it through `looksLikeCommandName`. If false, the function returns `false` (i.e., falls through to be sent to the model as a regular prompt). This is the minimal-blast-radius placement — `?` prefix and bare `/` autocomplete still flow through unchanged.
- `packages/cli/src/ui/utils/commandUtils.test.ts:93-159` — comprehensive unit coverage for `looksLikeCommandName`: valid names (`help`, `pr-review`, `gcp.deploy1`), MCP names with colons, hyphens/underscores/digits, and the rejection set including path separators (`api/endpoint`), non-ASCII (`接口`, `命令`), spaces, and special chars (`@`, `#`, `\`). The negative tests at `:142-154` are the load-bearing ones — they're what locks the behavior in against accidental future relaxation.
- `packages/cli/src/ui/utils/commandUtils.test.ts:184-202` — `isSlashCommand` regression tests covering `/api/apiFunction/接口的实现`, the long absolute path with mixed Chinese, `/var/log/syslog`, `/tmp/test.txt`, `/home/user/.config/settings.json`, `/etc/nginx/nginx.conf`. These are the actual reported failure modes from issue #1804 and they all assert `false` post-fix.
- `packages/cli/src/ui/hooks/slashCommandProcessor.test.ts:843-877` — integration test in the processor confirms `handleSlashCommand('/api/apiFunction/接口的实现')` and the long Desktop-path variant both return `false` (i.e., the command flow rejects them) rather than throwing or attempting a fuzzy command match.

## Reasoning

The bug class is real and user-visible: any user who tries to mention an absolute path in a prompt (`/Users/me/Desktop/foo`, `/etc/nginx/...`) was getting their input swallowed by the slash-command dispatcher and routed to a "command not found" path instead of being sent to the model. The fix is a two-layer defense — `isSlashCommand` is tightened to require a callable-looking first token, and `slashCommandProcessor` adds the same check at the dispatch site as belt-and-suspenders. Either layer alone would fix the reported issue; both together means a future refactor that loosens one won't silently regress the other.

The `looksLikeCommandName` character set is the load-bearing design decision. Allowing `:` and `.` is necessary (MCP tool names and extension-qualified commands), and rejecting non-ASCII is the right call for a slash-command grammar — Qwen Code's command registry uses ASCII identifiers exclusively, so any non-ASCII in the first token is by definition not a command. The Chinese-language test cases (`/接口实现`, `/文档 查看内容`) explicitly assert this and capture a class of input that was definitely getting mis-handled before.

Two minor things worth raising as nits: (1) the predicate doesn't currently allow `+` even though some MCP servers use it in tool names; if there's any usage of `+` in the wild it'll regress here. Worth a quick grep through `tool_registry` data before merge. (2) The PR body / commit message could benefit from explicitly listing issue #1804 as `Closes` so the auto-close fires on merge — the test names reference it but the metadata doesn't. Both are tiny. Otherwise this is a well-tested, narrowly-scoped UX fix that I'd happily merge.
