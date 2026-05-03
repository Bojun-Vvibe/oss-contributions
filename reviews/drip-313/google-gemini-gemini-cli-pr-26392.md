# google-gemini/gemini-cli PR #26392 — fix(windows): Resolve hangs, zombie processes, and improve subagent reliability

- Author: DovahkiinYuzuko
- Head SHA: `fa9a963a09ca26e3b9383feb14f4232e8b5e07b9`
- Verdict: **merge-after-nits**

## Rationale

Multi-faceted Windows reliability fix bundling several useful changes.
`packages/cli/src/commands/gemma/setup.ts:77-130` wraps the model download
in a 30s `AbortController` timeout — concrete fix for the hang where a
half-open TCP socket stalls setup forever; the `try/finally` clears the
timer correctly. `packages/cli/src/ui/components/LoadingIndicator.tsx:71`
makes the placeholder dynamic (`Thinking (Ns)...`) — small UX win.
`commandUtils.ts:45-60` trims input before slash detection so leading
whitespace stops swallowing commands. `activityLogger.ts:142,151-178`
raises buffer limit 10→50 and adds a `flushConsoleBuffer` writing to
`~/.gemini/logs/latest.log` with 512KB cap + 500-line trim. The flush
helper is defined but the diff snippet doesn't show the trigger site —
verify it's wired to the buffer-overflow path before merging. Earlier
rounds of this PR were `request-changes` (drip-295/303) at SHA
`4ff5a60f...`; the new SHA appears to be a different, smaller surface, so
worth re-evaluating. Address the flush-trigger verification + a unit test
for the 30s timeout, then ship.
