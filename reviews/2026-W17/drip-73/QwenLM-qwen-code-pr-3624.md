---
pr: 3624
repo: QwenLM/qwen-code
sha: 0d5221490d2e3c8d02ba4acef885f1051c6028a2
verdict: merge-after-nits
date: 2026-04-26
---

# QwenLM/qwen-code #3624 — fix(cli): add API Key option to `qwen auth` interactive menu

- **Author**: doudouOUC
- **Head SHA**: 0d5221490d2e3c8d02ba4acef885f1051c6028a2
- **Size**: ~578 diff lines, primarily a new `handleApiKeyAuth` flow plus the menu-option wiring.

## Scope

`qwen auth` previously offered OAuth and existing-credential paths but no interactive way to enter a raw API key without editing config. This PR adds an "API Key" option to the auth method picker that prompts for the key inline (masked), validates it with a probe call, and persists it to the standard auth-record location used by `AuthType.USE_OPENAI` / equivalent.

## Specific findings

- **Menu wiring** — the new option goes into the same `select`/`prompts` flow as the existing OAuth and credential entries. Confirm the label is consistently "API Key" across i18n bundles (English, zh-CN at minimum) — the qwen project ships both.
- **Input masking** — the prompt should use a password-style input (e.g., `prompts({ type: 'password' })` rather than `'text'`). If the diff uses a plain text input, the key will echo to the terminal during entry and end up in `~/.bash_history` / scrollback. Verify which prompt type is used.
- **Probe-call validation** — the right shape: enter key → probe a cheap endpoint (e.g., `models.list`) → on 401/403 reject with a clear message; on other errors (network, 5xx) accept the key but warn that validation could not complete. The worst pattern is rejecting on any non-200, which strands users behind a proxy or transient outage. Confirm which error classes block save vs. warn-and-save.
- **Persistence path** — the new path should write to the *same* file the existing `--api-key` flag and `OPENAI_API_KEY` env var read from, so the three entry points are interchangeable. If the interactive path writes to a different location, users who set a key interactively will be confused when env-var or flag override is not picked up.
- **File permissions** — `chmod 0600` (or `0644` if the file is genuinely non-secret) on the persisted credential file. On macOS/Linux, the auth file should be `0600`. Verify and add a regression test (`fs.statSync(path).mode & 0o777 === 0o600`).
- **Re-entry / overwrite** — what happens when the user runs `qwen auth` again with a key already saved? The flow should default-highlight the existing auth method and confirm before overwriting, not silently replace. If the diff replaces silently, surface a one-line "this will replace the existing API key saved at <path>" prompt.
- **Test coverage** — the diff is 578 lines. Hopefully a meaningful chunk is `handleApiKeyAuth.test.ts`. Required cases: (a) happy path (valid key → save → success message), (b) 401 from probe → no save, (c) network failure during probe → save with warning, (d) cancel mid-prompt (Ctrl+C) → no save, no partial file write.
- **Endpoint configurability** — if the probe call goes to a hardcoded `https://...alibabacloud.../v1/models`, users on a custom `OPENAI_BASE_URL` (Azure, self-hosted, proxy) will get a false 401. Read the configured base URL first and probe against that.
- **Logging hygiene** — confirm the key is never logged, even at debug level. A search of the diff for `console.log`/`logger.debug` adjacent to the key variable is sufficient.

## Risk

Low-medium. This is an additive UX option, not a refactor of the existing auth surface. The risks are concentrated in standard credential-handling pitfalls: input echo, file permissions, log leakage, and "interactive-vs-flag-vs-env" persistence consistency.

## Verdict

**merge-after-nits** — (1) confirm `prompts({ type: 'password' })` for masking; (2) confirm the persisted file is `0600`; (3) confirm the probe respects `OPENAI_BASE_URL` so non-default endpoints validate correctly; (4) reject only on auth-class errors (401/403), accept-with-warning on transient failures; (5) overwrite-confirm prompt when an API-key auth record already exists; (6) cover the four required test cases listed above; (7) i18n the new menu label. The feature itself is a real ergonomic win — the existing "edit your config to add a key" flow is hostile.
