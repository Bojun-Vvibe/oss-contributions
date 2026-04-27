# QwenLM/qwen-code #3577 — feat(skills): add tmux-real-user-testing skill for readable TUI test logs

- Author: pomelo-nwu
- Head SHA: `f6777d902d6e3b9fee2d945be441b36c70e5bdad`
- +401 / −0: new skill at `.qwen/skills/tmux-real-user-testing/SKILL.md` (259 lines) and helper script at `.qwen/skills/tmux-real-user-testing/scripts/tmux-real-user-log.sh` (142 lines)
- PR link: <https://github.com/QwenLM/qwen-code/pull/3577>

## Specifics

- The skill targets a real testing-workflow gap: React Ink TUIs emit ANSI escape sequences that make `tmux pipe-pane` raw logs unreadable. The skill prescribes `tmux capture-pane -p` snapshots labeled per step as the primary report-grade artifact, with `pipe-pane` demoted to "optional forensic". That distinction is correct (`capture-pane -p` reads the rendered cell grid; `pipe-pane` taps the raw byte stream).
- Helper script `tmux-real-user-log.sh:1-142` exposes six subcommands (`start`/`snapshot`/`send`/`type-submit`/`wait-for`/`finish`) with consistent positional-arg shape. `set -euo pipefail` at line 2 — correct strict-mode for a shell wrapper.
- `start` subcommand at `script:42-65` does three right things: (1) `tmux has-session -t "$session" 2>/dev/null` pre-check that fails fast on collision, (2) `printf 'export SESSION=%q\n'` with `%q` quoting for `eval` safety against scenario names containing spaces or shell metachars, (3) outputs three correlated env vars (`SESSION`/`OUTDIR`/`LOG`) that `eval` injects into the calling shell so subsequent commands don't need argument plumbing.
- `wait-for` at `script:99-119` is the most subtle command: it polls `capture-pane` into `current-pane.txt`, `grep -Eq`'s the regex, and on match dumps the matched pane to stdout and exits 0; on timeout (`attempts × sleep_seconds` default 60×2 = 120s) it dumps the last poll and exits 1. So the caller can pipe the output into a section header. Sensible defaults.
- `send` at `script:73-80` correctly uses a 0.15s sleep between keys to avoid Ink's input debounce swallowing characters — the SKILL.md prose at `:147-149` documents the same workaround for bulk-text fields. Behavior matches documentation.
- `type-submit` at `script:84-89` separates the typed text and the `Enter` keystroke with `sleep 0.5` — necessary because tmux's `send-keys "/auth Enter"` is interpreted as a single key sequence and Ink can race the rendering before processing the Enter. Correct fix.
- The skill description at `SKILL.md:3` includes Chinese-language trigger phrases ("用 tmux 做真实测试", "保存 tmux 日志", "像真实用户一样测试 Qwen", "生成可复查的 TUI 测试报告", "测试 slash command 交互") alongside English. This matches the bilingual user base of qwen-code and the existing skill-trigger conventions in `.qwen/skills/`.
- Output directory naming `tmp/<scenario>-tmux-YYYYMMDD-HHMMSS/` at `SKILL.md:48-55` plus the explicit "do not overwrite previous runs" rule preserves diagnostic value across iterative test runs.

## Concerns

- `script:33-36`'s `tmux new-session ... "$*"` interpolates the command via `$*` (space-joined), which means a scenario command containing words with shell-significant characters (e.g. `npm run dev -- --approval-mode 'yolo two words'`) won't survive the round-trip — tmux re-parses the joined string as a shell command. The SKILL.md examples never hit this case, but it's an unobvious pitfall worth documenting (or use `printf -v cmd '%q ' "$@"` instead).
- The `start` subcommand prints `export SESSION=… OUTDIR=… LOG=…` for `eval`, but doesn't `set +u`-guard the eventual user-shell consumption. If the caller already has `set -u` in their interactive shell, an unbound variable from a partial eval could be jarring. Probably fine since `eval` always sets all three together if the script reached the print path.
- `wait-for` reads `current-pane.txt` then `grep -Eq` immediately — there's a tiny race where capture-pane hasn't flushed yet. Empirically `capture-pane` is synchronous so this isn't a real bug, but a comment would help.
- The "Common pitfalls" section at `SKILL.md:243-260` is genuinely useful (mentions the `open <file>` macOS no-output-is-normal trap, the bulk-text Ink-input issue, the `pipe-pane` ANSI-garble issue) — exactly the institutional knowledge that's hard to rediscover.
- Skill scopes itself appropriately: the "When to use" section at `SKILL.md:30-41` says use this for TUI/dialog/keyboard/auth flows and explicitly defers to "headless JSON E2E" for tool-execution and model-API testing. That's the correct boundary.
- Minor: the example at `SKILL.md:225` shows `bash scripts/tmux-real-user-log.sh start ...` (relative path) but the canonical path is `.qwen/skills/tmux-real-user-testing/scripts/tmux-real-user-log.sh` (the path used at `:69-72`). The two example invocations should agree on which is canonical.

## Verdict

`merge-after-nits` — high-quality methodology skill grounded in a real distinction (`capture-pane` rendered grid vs `pipe-pane` raw bytes) with a well-engineered helper script (eval-safe quoting, session-name collision check, sensible default polling cadence). Two cleanups before merge: fix the path inconsistency in the auth-test example at `SKILL.md:225` to match `:69-72`, and either swap `$*` to a `%q`-quoted form at `script:33` or document the shell-quoting limitation. The Chinese-language trigger phrases lined up with qwen-code's bilingual norms is a nice touch.
