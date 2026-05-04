# Review: QwenLM/qwen-code #3649 — fix(lsp): expose status and startup diagnostics

- **PR**: https://github.com/QwenLM/qwen-code/pull/3649
- **Author**: yiliang114
- **Base**: `main`
- **Head SHA**: `06abcfd4a1e1ff2898ed56cbfafab98fcb532e0a`
- **Size**: +908 / −22 across 25 files

## Scope

Adds an LSP-status surface (snapshot in config + `/status` line + a new `/lsp` slash command), captures stdio startup diagnostics, refreshes the LSP user docs, and propagates the status into `aboutCommand` and `systemInfo` for support bundles.

## Substantive review

### `packages/cli/src/ui/commands/lspCommand.ts` (+114 new file)

- Correctly typed against `SlashCommand` and `CommandKind.BUILT_IN`.
- Three guard branches: no config, LSP not enabled, LSP enabled but no client. Each returns a localised user message via `t(...)`. Good — no hard crash paths.
- `statusIcon(status)` (line ~20) maps `READY|IN_PROGRESS|FAILED|NOT_STARTED` to emoji. Two minor concerns:
  1. The status string contract isn't enforced via a TS literal union here — if the server side emits e.g. `STARTING` (which the docs section mentions), it falls through to `❓`. Please align the casing/values with whatever `client.getServerStatus()` actually returns and use a `keyof typeof` to make drift a compile error.
  2. The docs (lsp.md) describe the snapshot status string `LSP: enabled, 1/1 ready`, but this command renders a markdown table with per-server rows. Nice — but the `/status` integration shown elsewhere in the diff should reuse `statusIcon` rather than maintaining a parallel mapping.
- The hint string at line ~63 — `"...grep '[LSP]' ~/.qwen/debug/latest"` — the `[LSP]` argument to `grep` will be interpreted as a character class matching `L`, `S`, or `P`. Use `grep -F '[LSP]'` (fixed-string) or escape the brackets. Same issue likely in the docs change.
- Line ~104: `lines.push(\`| ${server.name} | \`${cmd}\` | ${langs} | ${statusText} |\`);` — the inline backtick-quoted command will break the markdown table if `cmd` itself contains a `|` character. Replace `|` with `\\|` (or HTML-encode) when inserting cell values.

### `packages/cli/src/services/BuiltinCommandLoader.ts` (+2)

Registers `lspCommand` in the loader. Standard wiring; matching test added.

### `packages/cli/src/config/config.ts` (+16)

Adds an LSP-status snapshot exposed through the config object. Also adds a typed status enum (judging from the test diff). Looks correct, but I'd want to confirm:
- Snapshot is recomputed on each access vs. cached + invalidated on LSP lifecycle events. If cached, the `/lsp` command will display stale data after a server crashes mid-session.

### `packages/cli/src/ui/commands/aboutCommand.ts` (+1) and tests (+32)

Adds an `LSP` line to the about output. Test additions cover the rendering.

### `packages/cli/src/utils/systemInfo.ts` (+9 / +50 in two hunks)

Threads the LSP status into the extended system-info bundle used for bug reports. Also adds field surfacing in `systemInfoFields.ts`. This is the right place for it — operators triaging an LSP issue can ask the user to share their `/about` output.

### `docs/users/features/lsp.md` (+78 / −9)

- The reordered "Server Not Starting" checklist now leads with "Verify `--experimental-lsp` flag" — correct ordering, this is the most common cause of "nothing happens".
- New "Debugging" section is genuinely useful: explicit `rg`/`grep` commands, the four `[LSP]` / `[CONFIG]` / `[STATUS]` log entry shapes to look for, and a clangd troubleshooting subsection with `--check`/`--log=verbose` and `compile_commands.json` guidance. This is high-quality user-facing docs.
- The same `grep '[LSP]'` issue exists here: please use `grep -F '[LSP]'` or `rg '\[LSP\]'`.
- The reference to `~/.qwen/debug/latest` and `$QWEN_RUNTIME_DIR/debug/latest` is good — confirm both paths exist in current runtime code.

### Test additions

- `config.test.ts` (+38), `aboutCommand.test.ts` (+32), `systemInfo.test.ts` (+63), `core/.../config.test.ts` (+63 visible). Coverage looks proportionate.

## Verdict

**merge-after-nits** — substantive, useful diagnostic surface that will materially help LSP-related triage. Three concrete code nits before merge:
1. Use `grep -F '[LSP]'` (or `rg '\[LSP\]'`) in both `lspCommand.ts` and `lsp.md`.
2. Sanitize `|` in `server.name` / `server.command` / `langs` before inserting into the markdown table.
3. Tighten `statusIcon` parameter type to a string literal union derived from `client.getServerStatus()`'s declared return type so future status additions are caught at compile time.
