# sst/opencode #25800 — chore(i18n): complete Chinese translation for zh.ts files

- Head SHA: `1b1f53034105b4f76a88c2d3a6bb7b96e98816e6`
- Author: LifetimeVip
- Diff: +32 / −4 across 2 files (`packages/app/src/i18n/zh.ts`, `packages/ui/src/i18n/zh.ts`)
- Closes: #25604

## Verdict: `merge-as-is`

## What it does

Backfills 24 missing `app/` keys + 6 `ui/` keys + 1 corrected `desktop` label in the Simplified Chinese dictionary so the zh dict is at parity with the en source after recent feature adds (workspace settings, child-agent sessions, session review diffs, terminal/file-tree shell prefs, project navigation). Also fixes two stale English fallbacks: `command.prompt.mode.normal` had been left as the literal English `"Prompt"` and `ui.tool.webfetch` had been left as `"Webfetch"` — both now use the correct Chinese strings (`提示`, `网页获取`).

## Why this is fine to land

1. **Mechanical change with a verification step.** Author states "All 994 keys present in both EN and ZH TypeScript files. Verified via custom parser." The diff matches that — no key deletions, no type changes, no structural edits, just `satisfies Partial<Record<Keys, string>>` continues to hold since every added key is a string literal.
2. **Two genuine bug fixes folded in cleanly.** The `terminalFont.title` / `terminalFont.description` block at `packages/app/src/i18n/zh.ts:637-638` was previously inserted as raw English (`"Terminal Font"` / `"Customise the font used in the terminal"`) — the diff removes that and re-inserts the localized version (`终端字体` / `选择终端中使用的字体`) further down. Same fix-then-localize pattern for `command.prompt.mode.normal` (`"Prompt"` → `"提示"` at line 96) and `ui.tool.webfetch` (`"Webfetch"` → `"网页获取"` at `packages/ui/src/i18n/zh.ts:99`).
3. **Convention preserved.** Keyboard key names stay in English (`Shell`, `Glob`, `Grep`, `Git 更改`) per the existing zh.ts convention — no over-translation.
4. **Translation quality spot-check.** Sampled the more substantive entries:
   - `settings.workspace.enableTeam.delegateHint`: `"启用后，此设置将由工作区管理员管理。"` — accurate, natural.
   - `settings.general.row.shell.description`: `"选择终端使用的 shell。兼容的 shell 包括 bash、zsh、fish 等。"` — keeps `shell`/`bash`/`zsh`/`fish` in English (correct, these are command names), uses Chinese full-width comma + period (correct).
   - `session.child.promptDisabled`: `"子代理会话无法接收提示。"` — accurate translation of "Child agent sessions cannot receive prompts."
5. **No risk surface.** Pure data file, no runtime behavior change, no untranslated key deletions, no conflicts with the `Partial<Record<Keys, string>>` typing.

## Optional nit (not a blocker)

`ui.tool.webfetch` translates to `网页获取` ("web fetch / fetch web page") which is fine, but the sibling `ui.tool.websearch` is already `网络搜索` ("network search / web search"). For consistency the pair could be `网页` (web page) for both — `网页获取` + `网页搜索` — or `网络` for both. Not worth holding the PR; a follow-up tone-pass can normalize.

## Citations

- `packages/app/src/i18n/zh.ts:96` — `command.prompt.mode.normal` English-fallback fix
- `packages/app/src/i18n/zh.ts:637-638` — old hardcoded English `terminalFont` removal
- `packages/app/src/i18n/zh.ts:919-942` — 24 new app-module keys appended in stable alphabetic-ish order
- `packages/ui/src/i18n/zh.ts:99` — `ui.tool.webfetch` English-fallback fix
- `packages/ui/src/i18n/zh.ts:163-168` — 6 new sessionReview/sessionTurn keys
